# svnpatch by vIsHnU
svnpatch data new repository
A description of GIT's unidiff-compatible patch extensions, which we might
use for "svn patch" format. Copied from Augie Fackler's email
<http://subversion.tigris.org/ds/viewMessage.do?dsForumId=462&dsMessageId=2388540>.

Documentation for the format is pretty sparse. Here's an overview (now 
that I've written this out, it might be more like "everything you need 
to know to support this format other than specifics of how base85 
works"):

Copies and renames are both handled like this:
diff --git a/{oldname} b/{newname}
copy from {oldname}
copy to {newname}
{normal unidiff data here if there was a difference, only for how new 
is different from old}

(naturally, with copy instead of rename if that was the case}

New files:
diff --git a/alpha b/alpha
new file mode 100644
--- /dev/null
+++ b/alpha
@@ -0,0 +1,1 @@
+alpha

(the longer file mode is used for representing symlinks, they come in 
with mode 120000, and the contents are listed as *just* the path of 
the symlink, that is, svn's symlink format without the leading 'link ')

Removals:
diff --git a/README b/README
deleted file mode 100644
--- a/README
+++ /dev/null
@@ -1,77 +0,0 @@
{pile of unidiff data}

File mode changes:
diff --git a/README b/README
old mode 100644
new mode 100755
(I think it's normal for patch implementations to not care about the 
old mode, and just apply the new one blindly - the old mode I believe 
is more for human consumption or reverse-patching than anything else. 
Also, you don't really ever see any modes other than these two - the 
first is just "not executable", the second is executable. I've never 
*seen* any other modes for regular files).

Binaries (this is for literals, which I believe is the new binary in 
its entirety. I think there's a delta format which Mercurial does not 
support, per a "TODO: deltas" in the code):
index {old hash}..{new hash}
GIT binary patch
literal {file length}
{base85-encoded zlib-compressed data}

The hashes are either the nullid (40 0's) if there was (or is) no 
text, and are
sha1('blob %d\0%s' % (len(text), text, )) (using python syntax).
The base85 encoding seems to use a leading z to indicate this isn't 
the last line of data, and a leading e to indicate this is the last 
line of data. There's more to base85 than that, and I'd gladly take 
some time to write down how it works for those interested in 
reimplementing it. It appears at first glance to be the part that 
requries the most detailed documentation.

Supporting binary deltas is something that can obviously come later, 
since Mercurial doesn't appear to implement it at all right now. Note 
that with binary literals it does not appear that patch -R is 
possible. That's really a minor limitation, since you're never using 
this on non-version-controlled files anyway.

It should be possible to add your own hunk types to this format (once 
the base format is supported) that would allow you to transmit 
property modifications inline as well (other than symlink and 
executable, which are already handled sanely).

For those interested in some source code (GPLv2):
http://mercurial.selenic.com/hg/hg/file/ac02b43bc08a/mercurial/patch.py#l195
http://mercurial.selenic.com/hg/hg/file/ac02b43bc08a/mercurial/patch.py#l710
http://mercurial.selenic.com/hg/hg/file/ac02b43bc08a/mercurial/pure/base85.py
