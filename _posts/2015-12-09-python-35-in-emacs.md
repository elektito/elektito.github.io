---
layout: page
title: Python 3.5 in Emacs
---

> April 2016 Update: Native support for Python 3.5 has finally landed
> in Emacs development branch. You don't need this patch anymore.

Native co-routine support in Python 3.5 is really cool. So cool in
fact that it finally made me switch from Python 2. I never was a big
fan of Python 3, but that's another story. The point is I'm finally
ready to write some Python 3 code and I find out that my favorite
editor (Emacs) does not support the new functionality. I don't need
anything fancy really. I just need proper syntax coloring, proper
indentation and working navigation commands.

The indentation seems to be working fine in Emacs 24 already, but the
other two are not. The best I could find was the patch accompanying
[this message][1] on the Emacs mailing list. All I needed to do was to
get Emacs source code (`git clone
git://git.savannah.gnu.org/emacs.git`; there, I probably just spared
you one Google search!), make some minor modifications (see below) and
apply the patch (`git apply py35.diff`).

One small thing missing in the patch was proper coloring for the new
keyword `await`. Another problem was that the file locations are not
the same as in the git head. I've made the proper adjustments to make
it possible to apply the patch to the current master branch. You can
download the updated patch from the link below.

I should really thank Lele Gaifax from the Emacs mailing list. You
made my day stranger!

Download: [py35.diff][2]

 [1]: https://lists.gnu.org/archive/html/emacs-devel/2015-10/msg00558.html
 [2]: /public/extra/py35.diff
