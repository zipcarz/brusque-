brusque-
======
--- ureadahead-0.100.0.orig/conf/ureadahead-other.conf
+++ ureadahead-0.100.0/conf/ureadahead-other.conf
@@ -6,7 +6,7 
 
 description	"Read required files in advance (for other mountpoints)"
 
-start on mount DEVICE=[/UL]* MOUNTPOINT=/?*
+start on mounted DEVICE=[/UL]* MOUNTPOINT=/?*
 
 # Forks into the background both when reading from disk and when profiling
 # (HDD mode won't fork, but that's ok because we'll wait for it in spawned).
--- ureadahead-0.100.0.orig/man/ureadahead.8
+++ ureadahead-0.100.0/man/ureadahead.8
@@ -49,7 +49,7 @@
 has been enabled.
 
 This is ignored when reading on rotational hard drives, since it's
-important for performance reasons not to content with other processes
+important for performance reasons not to contend with other processes
 for I/O.
 .\"
 .TP
@@ -115,7 +115,7 @@
 .\"
 .SH BUGS
 Report bugs at 
-.RB < https://launchpad.net/ureadahead/+bugs >
+.RB < https://launchpad.net/ubuntu/+source/ureadahead/+bugs >
 .\"
 .SH COPYRIGHT
 Copyright \(co 2009 Canonical Ltd.
--- ureadahead-0.100.0.orig/src/Makefile.am
+++ ureadahead-0.100.0/src/Makefile.am
@@ -22,6 +22,7 @@
 	file.c file.h \
 	errors.h
 ureadahead_LDADD = \
+	-lrt \
 	$(NIH_LIBS) \
 	$(BLKID_LIBS) \
 	$(EXT2FS_LIBS) \
--- ureadahead-0.100.0.orig/src/trace.c
+++ ureadahead-0.100.0/src/trace.c
@@ -117,17 +117,20 @@
        int timeout)
 {
 	int                 dfd;
+	FILE                *fp;
 	int                 unmount = FALSE;
 	int                 old_sys_open_enabled = 0;
 	int                 old_open_exec_enabled = 0;
 	int                 old_uselib_enabled = 0;
 	int                 old_tracing_enabled = 0;
+	int                 old_buffer_size_kb = 0;
 	struct sigaction    act;
 	struct sigaction    old_sigterm;
 	struct sigaction    old_sigint;
 	struct timeval      tv;
 	nih_local PackFile *files = NULL;
 	size_t              num_files = 0;
+	size_t              num_cpus = 0;
 
 	/* Mount debugfs if not already mounted */
 	dfd = open (PATH_DEBUGFS "/tracing", O_RDONLY | O_NOATIME);
@@ -148,6 +151,28 @@
 		unmount = TRUE;
 	}
 
+	/*
+	 * Count the number of CPUs, default to 1 on error. 
+	 */
+	fp = fopen("/proc/cpuinfo", "r");
+	if (fp) {
+		int line_size=1024;
+		char *processor="processor";
+		char *line = malloc(line_size);
+		if (line) {
+			num_cpus = 0;
+			while (fgets(line,line_size,fp) != NULL) {
+				if (!strncmp(line,processor,strlen(processor)))
+					num_cpus++;
+			}
+			free(line);
+			nih_message("Counted %d CPUs\n",num_cpus);
+		}
+		fclose(fp);
+	}
+	if (!num_cpus)
+		num_cpus = 1;
+
 	/* Enable tracing of open() syscalls */
 	if (set_value (dfd, "events/fs/do_sys_open/enable",
 		       TRUE, &old_sys_open_enabled) < 0)
@@ -165,7 +190,7 @@
 
 		old_uselib_enabled = -1;
 	}
-	if (set_value (dfd, "buffer_size_kb", 128000, NULL) < 0)
+	if (set_value (dfd, "buffer_size_kb", 8192/num_cpus, &old_buffer_size_kb) < 0)
 		goto error;
 	if (set_value (dfd, "tracing_enabled",
 		       TRUE, &old_tracing_enabled) < 0)
@@ -226,6 +251,13 @@
 	if (read_trace (NULL, dfd, "trace", &files, &num_files) < 0)
 		goto error;
 
+	/*
+	 * Restore the trace buffer size (which has just been read) and free
+	 * a bunch of memory.
+	 */
+	if (set_value (dfd, "buffer_size_kb", old_buffer_size_kb, NULL) < 0)
+		goto error;
+
 	/* Unmount the temporary debugfs mount if we mounted it */
 	if (close (dfd)) {
 		nih_error_raise_system ();
--- ureadahead-0.100.0.orig/src/Makefile.in
+++ ureadahead-0.100.0/src/Makefile.in
@@ -295,6 +295,7 @@
 	errors.h
 
 ureadahead_LDADD = \
+	-lrt \
 	$(NIH_LIBS) \
 	$(BLKID_LIBS) \
 	$(EXT2FS_LIBS) \
--- ureadahead-0.100.0.orig/src/pack.c
+++ ureadahead-0.100.0/src/pack.c
@@ -163,6 +163,7 @@
 		unsigned int maj;
 		unsigned int min;
 		char *       mount;
+		struct stat  statbuf;
 
 		/* mount ID */
 		ptr = strtok_r (line, " \t\n", &saveptr);
@@ -185,14 +186,6 @@
 			continue;
 		}
 
-		/* Check whether this is the right device */
-		if ((sscanf (device, "%d:%d", &maj, &min) < 2)
-		    || (maj != major (dev))
-		    || (min != minor (dev))) {
-			nih_free (line);
-			continue;
-		}
-
 		/* root */
 		ptr = strtok_r (NULL, " \t\n", &saveptr);
 		if (! ptr) {
@@ -206,6 +199,12 @@
 			nih_free (line);
 			continue;
 		}
+
+		/* Check whether this is the right device */
+		if (stat (mount, &statbuf) || statbuf.st_dev != dev) {
+			nih_free (line);
+			continue;
+		}
 
 		/* Done, convert the mountpoint to a pack filename */
 		if (fclose (fp) < 0)
--- ureadahead-0.100.0.orig/src/values.c
+++ ureadahead-0.100.0/src/values.c
@@ -57,7 +57,7 @@
 	if (fd < 0)
 		nih_return_system_error (-1);
 
-	len = read (fd, buf, sizeof buf);
+	len = read (fd, buf, sizeof(buf) - 1);
 	if (len < 0) {
 		nih_error_raise_system ();
 		close (fd);
@@ -90,7 +90,7 @@
 		nih_return_system_error (-1);
 
 	if (oldvalue) {
-		len = read (fd, buf, sizeof buf);
+		len = read (fd, buf, sizeof(buf) - 1);
 		if (len < 0) {
 			nih_error_raise_system ();
 			close (fd);
--- ureadahead-0.100.0.orig/debian/copyright
+++ ureadahead-0.100.0/debian/copyright
@@ -0,0 +1,17 @@
+This is the Ubuntu package of ureadahead.
+
+Copyright © 2009 Canonical Ltd.
+
+Licence:
+
+This program is free software; you can redistribute it and/or modify
+it under the terms of the GNU General Public License version 2, as
+published by the Free Software Foundation.
+
+This program is distributed in the hope that it will be useful, but
+WITHOUT ANY WARRANTY; without even the implied warranty of
+MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the GNU
+General Public License for more details.
+
+On Ubuntu systems, the complete text of the GNU General Public License
+can be found in ‘/usr/share/common-licenses/GPL-2’.
--- ureadahead-0.100.0.orig/debian/compat
+++ ureadahead-0.100.0/debian/compat
@@ -0,0 +1 @@
+7
--- ureadahead-0.100.0.orig/debian/ureadahead.postrm
+++ ureadahead-0.100.0/debian/ureadahead.postrm
@@ -0,0 +1,61 @@
+#!/bin/sh -e
+# This script can be called in the following ways:
+#
+# After the package was removed:
+#	<postrm> remove
+#
+# After the package was purged:
+#	<postrm> purge
+#
+# After the package was upgraded:
+#	<old-postrm> upgrade <new-version>
+# if that fails:
+#	<new-postrm> failed-upgrade <old-version>
+#
+#
+# After all of the packages files have been replaced:
+#	<postrm> disappear <overwriting-package> <version>
+#
+#
+# If preinst fails during install:
+#	<new-postrm> abort-install
+#
+# If preinst fails during upgrade of removed package:
+#	<new-postrm> abort-install <old-version>
+#
+# If preinst fails during upgrade:
+#	<new-postrm> abort-upgrade <old-version>
+
+
+# Remove pack file
+purge_files()
+{
+    if [ -f /var/lib/ureadahead/pack ]; then
+	rm -f /var/lib/ureadahead/pack /var/lib/ureadahead/*.pack || true
+	rmdir /var/lib/ureadahead || true
+    fi
+}
+
+
+case "$1" in
+    remove)
+	;;
+
+    purge)
+	purge_files
+	;;
+
+    upgrade|failed-upgrade|disappear)
+	;;
+
+    abort-install|abort-upgrade)
+	;;
+
+    *)
+	echo "$0 called with unknown argument \`$1'" 1>&2
+	exit 1
+	;;
+esac
+
+#DEBHELPER#
+exit 0
--- ureadahead-0.100.0.orig/debian/ureadahead.dirs
+++ ureadahead-0.100.0/debian/ureadahead.dirs
@@ -0,0 +1,3 @@
+usr/share/apport/package-hooks
+var/lib/ureadahead
+var/lib/ureadahead/debugfs
--- ureadahead-0.100.0.orig/debian/control
+++ ureadahead-0.100.0/debian/control
@@ -0,0 +1,24 @@
+Source: ureadahead
+Section: admin
+Priority: required
+Maintainer: Scott James Remnant <scott@ubuntu.com>
+Standards-Version: 3.9.1
+Build-Depends: debhelper (>= 7.3.15ubuntu3), pkg-config (>= 0.22), libnih-dev (>= 1.0.0), libblkid-dev (>= 2.16), e2fslibs-dev (>= 1.41)
+
+Package: ureadahead
+Architecture: any
+Depends: ${shlibs:Depends}, ${misc:Depends}, upstart (>= 0.6.0)
+Conflicts: readahead
+Provides: readahead
+Replaces: readahead
+Description: Read required files in advance
+ über-readahead is used during boot to read files in advance of when
+ they are needed such that they are already in the page cache,
+ improving boot performance.
+ .
+ Its data files are regenerated on the first boot after install, and
+ either monthly thereafter or when packages with init scripts or
+ configs are installed or updated.
+ .
+ ureadahead requires a kernel patch included in the Ubuntu kernel.
+
--- ureadahead-0.100.0.orig/debian/ureadahead.postinst
+++ ureadahead-0.100.0/debian/ureadahead.postinst
@@ -0,0 +1,44 @@
+#!/bin/sh -e
+# This script can be called in the following ways:
+#
+# After the package was installed:
+#	<postinst> configure <old-version>
+#
+# After a trigger is activated:
+#       <postinst> triggered <trigger-name>...
+#
+#
+# If prerm fails during upgrade or fails on failed upgrade:
+#	<old-postinst> abort-upgrade <new-version>
+#
+# If prerm fails during deconfiguration of a package:
+#	<postinst> abort-deconfigure in-favour <new-package> <version>
+#	           removing <old-package> <version>
+#
+# If prerm fails during replacement due to conflict:
+#	<postinst> abort-remove in-favour <new-package> <version>
+
+
+case "$1" in
+    configure)
+	;;
+
+    triggered)
+	# Force a reprofile
+	if [ -f /var/lib/ureadahead/pack ]; then
+	    echo "ureadahead will be reprofiled on next reboot"
+	    rm -f /var/lib/ureadahead/pack /var/lib/ureadahead/*.pack
+	fi
+	;;
+
+    abort-upgrade|abort-deconfigure|abort-remove)
+	;;
+
+    *)
+	echo "$0 called with unknown argument \`$1'" 1>&2
+	exit 1
+	;;
+esac
+
+#DEBHELPER#
+exit 0
--- ureadahead-0.100.0.orig/debian/rules
+++ ureadahead-0.100.0/debian/rules
@@ -0,0 +1,28 @@
+#!/usr/bin/make -f
+%:
+	dh $@
+
+
+CFLAGS = -Wall -g -fstack-protector -fPIE
+LDFLAGS = -Wl,-z,relro -Wl,-z,now -pie
+
+# Disable optimisations if noopt found in $DEB_BUILD_OPTIONS
+ifneq (,$(findstring noopt,$(DEB_BUILD_OPTIONS)))
+	CFLAGS += -O0
+	LDFLAGS += -Wl,-O0
+else
+	CFLAGS += -Os
+	LDFLAGS += -Wl,-O1
+endif
+
+override_dh_auto_configure:
+	dh_auto_configure -- CFLAGS="$(CFLAGS)" LDFLAGS="$(LDFLAGS)" \
+		--exec-prefix=
+
+
+override_dh_install:
+	dh_install
+	install -m 644 debian/ureadahead.apport \
+		debian/ureadahead/usr/share/apport/package-hooks/ureadahead.py
+
+version := $(shell sed -e '1{;s|^ureadahead (\(.*\))\ .*|\1|;q;}' debian/changelog)
--- ureadahead-0.100.0.orig/debian/ureadahead.triggers
+++ ureadahead-0.100.0/debian/ureadahead.triggers
@@ -0,0 +1,2 @@
+interest /etc/init
+interest /etc/init.d
--- ureadahead-0.100.0.orig/debian/changelog
+++ ureadahead-0.100.0/debian/changelog
@@ -0,0 +1,177 @@
+ureadahead (0.100.0-11) natty; urgency=low
+
+  * src/trace.c: leave room for string termination on reads (LP: #485194).
+  * man/ureadahead.8: fix typo and update bug reporting URL (LP: #697770).
+  * debian/rules: don't bother with /var/lib/ureadahead mode.
+
+ -- Kees Cook <kees@ubuntu.com>  Wed, 16 Mar 2011 17:19:01 -0700
+
+ureadahead (0.100.0-10) natty; urgency=low
+
+  * Install /var/lib/ureadahead mode 0700 so other users cannot see
+    the debugfs mount point.
+
+ -- Kees Cook <kees@ubuntu.com>  Tue, 22 Feb 2011 12:13:22 -0800
+
+ureadahead (0.100.0-9) natty; urgency=low
+
+  [ Bilal Akhtar ]
+  * Removed sreadahead transitional package and its postinst, postrm,
+    preinst and install files. (LP: #545596)
+  * Passed --sourcedir argument to dh_install
+  * Removed dh_gencontrol calls for package sreadahead.
+
+  [ Martin Pitt ]
+  * src/Makefile.{am,in}: Add missing -lrt, as pack.c uses clock_gettime.
+    Fixes building with gcc 4.5.
+  * debian/rules: Revert --sourcedir passing, it's not necessary.
+  * debian/rules: Don't install apport hook as executable.
+  * debian/copyright: Point to versioned GPL-2 file.
+  * debian/control: Bump Standards-Version to 3.9.1.
+
+ -- Martin Pitt <martin.pitt@ubuntu.com>  Fri, 26 Nov 2010 12:29:40 +0100
+
+ureadahead (0.100.0-8) maverick; urgency=low
+
+  * Decrease the buffer size to just 8MB, after much testing we don't
+    need much more than this since it will be limited by the size of the
+    page cache anyway.
+
+    This is in lieu of a new version of ureadahead for Maverick, which
+    while work is ongoing, isn't ready for shipping at this time.
+    LP: #600359.
+
+ -- Scott James Remnant <scott@ubuntu.com>  Mon, 20 Sep 2010 18:34:31 +0100
+
+ureadahead (0.100.0-7) maverick; urgency=low
+
+  * Count the number of CPUs and divide buffer_size_kb by the number of CPUs.
+    Users should watch for instances of "mmiotrace has lost events" in dmesg to make
+    sure the trace buffers are not too small. The original value for
+    buffer_size_kb was chosen somewhat arbitrarily. Empirical testing
+    has shown that its large enough, so we don't actually know where the lower
+    boundary lies.
+    -LP: #491943
+
+ -- Tim Gardner <tim.gardner@canonical.com>  Fri, 20 Aug 2010 12:19:31 -0600
+
+ureadahead (0.100.0-6) maverick; urgency=low
+
+  * Restore buffer_size_kb upon exit, but do it _after_
+    the trace buffer has been read. This frees the memory
+    consumed by the trace operation (which can be a lot).
+    -LP: #501715
+
+ -- Tim Gardner <tim.gardner@canonical.com>  Thu, 22 Jul 2010 04:04:36 -0600
+
+ureadahead (0.100.0-5) maverick; urgency=low
+
+  * src/pack.c: Amend mount point detection logic to stat the mount point
+    instead of just comparing major/minor versions with /proc/self/mountinfo
+    (LP: #570014).
+
+ -- Chow Loong Jin <hyperair@ubuntu.com>  Fri, 25 Jun 2010 13:14:54 +0100
+
+ureadahead (0.100.0-4.1) lucid; urgency=low
+
+  * Revert previous upload; had forgotten that the sreadahead package
+    contains all the clean-up stuff so we want to keep it for the
+    release upgrade after all.
+
+ -- Scott James Remnant <scott@ubuntu.com>  Wed, 17 Feb 2010 12:57:00 +0000
+
+ureadahead (0.100.0-4) lucid; urgency=low
+
+  * debian/control: drop sreadahead migration package; dist-upgrade users
+    will have ureadahead installed by the standard meta-packages. 
+
+ -- Scott James Remnant <scott@ubuntu.com>  Wed, 17 Feb 2010 12:14:09 +0000
+
+ureadahead (0.100.0-3) lucid; urgency=low
+
+  * conf/ureadahead-other.conf: Change from "on mount" to "on mounted",
+    the former didn't work anyway. 
+
+ -- Scott James Remnant <scott@ubuntu.com>  Mon, 21 Dec 2009 23:20:02 +0000
+
+ureadahead (0.100.0-2) lucid; urgency=low
+
+  * Put an all-important "--" in the dh_auto_configure invocation so that
+    ureadahead is installed into the right path (/sbin)
+
+ -- Scott James Remnant <scott@ubuntu.com>  Tue, 01 Dec 2009 02:25:50 +0000
+
+ureadahead (0.100.0-1) lucid; urgency=low
+
+  * New upstream release:
+    - Use external libnih
+
+  * debian/control: Add build-dependency on libnih-dev
+  * debian/rules: Fix installation of apport hook.
+
+ -- Scott James Remnant <scott@ubuntu.com>  Sun, 29 Nov 2009 15:24:15 +0000
+
+ureadahead (0.90.3-2) karmic-proposed; urgency=low
+
+  * über-readahead is a replacement for sreadahead that should
+    significantly improve boot performance on rotational hard drives,
+    especially those that had regressed in performance from jaunty to
+    karmic.
+
+    It does this by pre-loading such things as ext2/3/4 inodes and opening
+    files in as logical order as possible before loading all blocks in one
+    pass across the disk.
+
+    On SSD, this behaves much as sreadahead used to, replacing that package
+    with slightly improved tracing code.
+
+    This requires the kernel package also found in karmic-proposed.
+
+    LP: #432089.
+
+ -- Scott James Remnant <scott@ubuntu.com>  Mon, 09 Nov 2009 18:38:51 +0000
+
+ureadahead (0.90.3-1) karmic; urgency=low
+
+  * New upstream release:
+    - Move ext2fs inode group lookup into the tracing stage, storing the
+      groups to preload in the pack, rather than spending time on normal
+      boots working it out.
+    - Open files in order of inode group (or inode number on non-ext2fs),
+      which seems to give a benefit in load time and certainly produces
+      better blktrace output.
+    - Increase the "too old" check from a month to a year.
+    - Fix dump of zero-byte files to not claim a single page.
+    - Fix unhandled error output when given an unknown pack file.
+    - Don't call ureadhead for the root filesystem twice on boot (the second
+      time should only take a few ms, but that's still time)
+    - Consider exit status 4 (no pack file for given mount point) normal.
+    - Make uselib tracing optional.
+
+ -- Scott James Remnant <scott@ubuntu.com>  Thu, 05 Nov 2009 15:10:06 +0000
+
+ureadahead (0.90.2-1) karmic; urgency=low
+
+  * New upstream release:
+    - improved SSD mode
+    - inode group preload threshold configurable by environment variable
+    - default inode group preload threshold changed to 16, because random
+      stabbing in the dark suggested it was a good number
+    - add a job that profiles extra mountpoints
+
+  * Remove /etc/cron.monthly/sreadahead too.
+  * Add an apport hook to attach a dump of the packs.
+
+ -- Scott James Remnant <scott@ubuntu.com>  Thu, 29 Oct 2009 18:14:51 +0000
+
+ureadahead (0.90.1-1) karmic; urgency=low
+
+  * Bug fixes.
+
+ -- Scott James Remnant <scott@ubuntu.com>  Thu, 29 Oct 2009 02:13:38 +0000
+
+ureadahead (0.90.0-1) karmic; urgency=low
+
+  * Initial release to ubuntu-boot PPA.
+
+ -- Scott James Remnant <scott@ubuntu.com>  Thu, 29 Oct 2009 01:01:42 +0000
--- ureadahead-0.100.0.orig/debian/ureadahead.apport
+++ ureadahead-0.100.0/debian/ureadahead.apport
@@ -0,0 +1,15 @@
+'''apport package hook for ureadahead
+
+(c) 2009 Canonical Ltd.
+Author: Scott James Remnant <scott@ubuntu.com>
+'''
+
+import os
+import apport.hookutils
+
+def add_info(report):
+    for f in os.listdir('/var/lib/ureadahead'):
+        if f == 'pack':
+            report['PackDump'] = apport.hookutils.command_output(['ureadahead', '--dump'])
+        elif f.endswith('.pack'):
+            report['PackDump'+f[:-5].title()] = apport.hookutils.command_output(['ureadahead', '--dump'])
