---
layout: post
title: 关于Gitignore的规则
date: 2014-01-22 11:00
tags: git, gitignore
---


关于`.gitignore`的语法规则我一直没有很明白，今天决心要弄懂。

下面的是**Manual Page**的说明：

> A blank line matches no files, so it can serve as a separator for readability.
> 
> A line starting with # serves as a comment. Put a backslash ("\") in front of the first hash for patterns that begin with a hash.
> 
> An optional prefix "!" which negates the pattern; any matching file excluded by a previous pattern will become included again. If a negated pattern matches, this will override lower precedence patterns sources. Put a backslash ("\") in front of the first "!" for patterns that begin with a literal "!", for example, "\!important!.txt".
> 
> If the pattern ends with a slash, it is removed for the purpose of the following description, but it would only find a match with a directory. In other words, foo/ will match a directory foo and paths underneath it, but will not match a regular file or a symbolic link foo (this is consistent with the way how pathspec works in general in git).
> 
> If the pattern does not contain a slash /, git treats it as a shell glob pattern and checks for a match against the pathname relative to the location of the .gitignore file (relative to the toplevel of the work tree if not from a .gitignore file).
> 
> Otherwise, git treats the pattern as a shell glob suitable for consumption by fnmatch(3) with the FNM_PATHNAME flag: wildcards in the pattern will not match a / in the pathname. For example, "Documentation/*.html" matches "Documentation/git.html" but not "Documentation/ppc/ppc.html" or "tools/perf/Documentation/perf.html".
> 
> A leading slash matches the beginning of the pathname. For example, "/*.c" matches "cat-file.c" but not "mozilla-sha1/sha1.c".

我仔细读了半天，发现还是没弄懂，还是亲自做实验吧。

其实没弄懂的主要是第4，5，6条，一个个解决吧。

## If the pattern ends with a slash

原文这样：

> If the pattern ends with a slash, it is removed for the purpose of the following description, but it would only find a match with a directory. In other words, foo/ will match a directory foo and paths underneath it, but will not match a regular file or a symbolic link foo (this is consistent with the way how pathspec works in general in git).

正常情况:

    Carl-MBPR ➜  testgitignore git:(master) ✗ cat .gitignore
    foo/
    Carl-MBPR ➜  testgitignore git:(master) ✗ tree -a -L 1
    .
    ├── .git
    ├── .gitignore
    └── foo
    Carl-MBPR ➜  testgitignore git:(master) ✗ git status
    # On branch master
    # Untracked files:
    #   (use "git add <file>..." to include in what will be committed)
    #
    #   foo
    nothing added to commit but untracked files present (use "git add" to track)
    Carl-MBPR ➜  testgitignore git:(master) ✗ rm foo
    Carl-MBPR ➜  testgitignore git:(master) mkdir foo
    Carl-MBPR ➜  testgitignore git:(master) touch foo/file
    Carl-MBPR ➜  testgitignore git:(master) tree -L 2
    .
    └── foo
        └── file

    1 directory, 1 file
    Carl-MBPR ➜  testgitignore git:(master) git status
    # On branch master
    nothing to commit, working directory clean

子目录中的表现：

    Carl-MBPR ➜  testgitignore git:(master) mkdir somefolder
    Carl-MBPR ➜  testgitignore git:(master) touch somefolder/foo
    Carl-MBPR ➜  testgitignore git:(master) ✗ git status
    # On branch master
    # Untracked files:
    #   (use "git add <file>..." to include in what will be committed)
    #
    #   somefolder/
    nothing added to commit but untracked files present (use "git add" to track)
    Carl-MBPR ➜  testgitignore git:(master) ✗ rm somefolder/foo
    Carl-MBPR ➜  testgitignore git:(master) mkdir somefolder/foo
    Carl-MBPR ➜  testgitignore git:(master) touch somefolder/foo/file
    Carl-MBPR ➜  testgitignore git:(master) git status
    # On branch master
    nothing to commit, working directory clean

**结论**：这里的规则是在所有目录层级中都有效的。

## If the pattern does not contain a slash

> If the pattern does not contain a slash /, git treats it as a shell glob pattern and checks for a match against the pathname relative to the location of the .gitignore file (relative to the toplevel of the work tree if not from a .gitignore file).

实验：

    Carl-MBPR ➜  testgitignore git:(master) cat .gitignore
    foo

正常情况：

    Carl-MBPR ➜  testgitignore git:(master) touch foo
    Carl-MBPR ➜  testgitignore git:(master) git status
    # On branch master
    nothing to commit, working directory clean

    Carl-MBPR ➜  testgitignore git:(master) rm foo
    Carl-MBPR ➜  testgitignore git:(master) mkdir foo
    Carl-MBPR ➜  testgitignore git:(master) touch foo/file
    Carl-MBPR ➜  testgitignore git:(master) tree
    .
    └── foo
        └── file

    1 directory, 1 file
    Carl-MBPR ➜  testgitignore git:(master) git status
    # On branch master
    nothing to commit, working directory clean

子目录中表现：

    Carl-MBPR ➜  testgitignore git:(master) mkdir somefolder
    Carl-MBPR ➜  testgitignore git:(master) touch somefolder/foo
    Carl-MBPR ➜  testgitignore git:(master) tree
    .
    ├── foo
    │   └── file
    └── somefolder
        └── foo

    2 directories, 2 files
    Carl-MBPR ➜  testgitignore git:(master) git status
    # On branch master
    nothing to commit, working directory clean

**结论**：这里的规则也是对所有子目录有效。

## Otherwise, git treats the pattern as a shell glob suitable for consumption by fnmatch(3) with the FNM_PATHNAME flag

原文:

> Otherwise, git treats the pattern as a shell glob suitable for consumption by fnmatch(3) with the FNM_PATHNAME flag: wildcards in the pattern will not match a / in the pathname. For example, "Documentation/*.html" matches "Documentation/git.html" but not "Documentation/ppc/ppc.html" or "tools/perf/Documentation/perf.html".

实验：

    Carl-MBPR ➜  testgitignore git:(master) cat .gitignore
    foo/*.file

正常情况：

    Carl-MBPR ➜  testgitignore git:(master) mkdir foo
    Carl-MBPR ➜  testgitignore git:(master) touch foo/aa
    Carl-MBPR ➜  testgitignore git:(master) ✗ git status
    # On branch master
    # Untracked files:
    #   (use "git add <file>..." to include in what will be committed)
    #
    #   foo/
    nothing added to commit but untracked files present (use "git add" to track)
    Carl-MBPR ➜  testgitignore git:(master) ✗ mv foo/aa foo/aa.file
    Carl-MBPR ➜  testgitignore git:(master) tree
    .
    └── foo
        └── aa.file

    1 directory, 1 file
    Carl-MBPR ➜  testgitignore git:(master) git status
    # On branch master
    nothing to commit, working directory clean

子目录中表现：

    Carl-MBPR ➜  testgitignore git:(master) mkdir somefolder
    Carl-MBPR ➜  testgitignore git:(master) mkdir somefolder/foo
    Carl-MBPR ➜  testgitignore git:(master) touch somefolder/foo/some.file
    Carl-MBPR ➜  testgitignore git:(master) ✗ tree
    .
    ├── foo
    │   └── aa.file
    └── somefolder
        └── foo
            └── some.file

    3 directories, 2 files
    Carl-MBPR ➜  testgitignore git:(master) ✗ git status
    # On branch master
    # Untracked files:
    #   (use "git add <file>..." to include in what will be committed)
    #
    #   somefolder/
    nothing added to commit but untracked files present (use "git add" to track)

**结论**：该类型规则不会在子目录中生效。

## 总结

1. 当规则是简单文件（目录）名的匹配的话，会在当前目录和子目录中生效。
2. 如果规则中包含的路径（即使不是绝对路径），该规则都只会在当前目录生效。
3. 可以使用`/foo`的形式来从项目根目录开始匹配。
