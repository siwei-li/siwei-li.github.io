---
layout: post
title: Windows 下 gVim 使用 python3 的智能提示
date: 2012-09-17 00:58
tags: [python, python3, vim, autocomplete, gvim]
---

之前一直以为windows下的gvim是不支持python的只能提示(`Ctrl-x Ctrl-o`)的，因为每次执行都会被提示需要在编译时加上`+python`。今天又上网搜了下该问题，发现貌似并不是这个问题。

在gvim中执行`:version`输出如下

    VIM - Vi IMproved 7.3 (2010 Aug 15, compiled Oct 27 2010 17:59:02)
    MS-Windows 32 位图形界面版本 带 OLE 支持
    包含补丁: 1-46
    编译者 Bram@KIBAALE
    大型版本 带图形界面。  可使用(+)与不可使用(-)的功能:
    +arabic +autocmd +balloon_eval +browse ++builtin_terms +byte_offset +cindent
    +clientserver +clipboard +cmdline_compl +cmdline_hist +cmdline_info +comments
    +conceal +cryptv +cscope +cursorbind +cursorshape +dialog_con_gui +diff
    +digraphs -dnd -ebcdic +emacs_tags +eval +ex_extra +extra_search +farsi
    +file_in_path +find_in_path +float +folding -footer +gettext/dyn -hangul_input
    +iconv/dyn +insert_expand +jumplist +keymap +langmap +libcall +linebreak
    +lispindent +listcmds +localmap -lua +menu +mksession +modify_fname +mouse
    +mouseshape +multi_byte_ime/dyn +multi_lang -mzscheme +netbeans_intg +ole
    -osfiletype +path_extra +perl/dyn +persistent_undo -postscript +printer
    -profile +python/dyn +python3/dyn +quickfix +reltime +rightleft +ruby/dyn
    +scrollbind +signs +smartindent -sniff +startuptime +statusline -sun_workshop
    +syntax +tag_binary +tag_old_static -tag_any_white +tcl/dyn -tgetent
    -termresponse +textobjects +title +toolbar +user_commands +vertsplit
    +virtualedit +visual +visualextra +viminfo +vreplace +wildignore +wildmenu
    +windows +writebackup -xfontset -xim -xterm_save +xpm_w32
    ......

里面有`+python/dyn +python3/dyn`，说明其本身是支持python的。但是在gvim中执行`:echo has("python")`的时候返回为零，并且`:py <code>`也没法执行。

在网上搜索的时候发现有人提到了gvim和python的安装顺序。我看了下，发现我是先安装了gvim-windows再安装了python3-windows的，觉得有可能是这个问题。于是把gvim卸了重新安装，再执行`:echo has("python")`还是返回零。

又想到是不是由于我安装的是python3？于是就试了下`:echo has("python3")`，果然这次返回了1，`:py3 <code>`也是正常的。所以这里我也不确定是不是和安装顺序有关，因为之前从来没有试过`:py3`。

重新打开一个python代码，用`Ctrl-x Ctrl-o`发现还是不行，一样的提示

    Error: Required vim compiled with +python
    E117: Unknown function: pythoncomplete#Complete

不过有了以上的经历我猜到应该是vim还是去找python2了，问题可能是`pythoncomplete#Complete`这里。在gvim安装目录里面搜了一下`python`，发现有`pythoncomplete.vim`这个包和`python3complete.vim`这个包。所以应该是文件类型处理里面指定了使用`pythoncomplete`而不是`python3complete`。

打开gvim目录里面的ftplugin/python.vim，果然找到了

    setlocal omnifunc=pythoncomplete#Complete

改成

    setlocal omnifunc=python3complete#Complete

这样就可以正常使用了。
