---
layout: post
title: Shell Terminfo工具tput
comments: true
tags: [shell]
header-img: img/post-bg-universe.jpg
---

## Terminfo是什么？
  UNIX 系统上的 terminfo数据库用于定义终端和打印机的属性及功能，包括各设备（例如，终端和打印机）的行数和列数以及要发送至该设备的文本的属性。UNIX 中的几个常用程序都依赖 terminfo数据库提供这些属性以及许多其他内容，其中包括 vi 和 emacs 编辑器以及 curses 和 man 程序。通过命令infocmp可查看支持的属性。
  ```bash
  ~$ infocmp
#	Reconstructed via infocmp from file: /usr/share/terminfo/x/xterm
xterm|xterm terminal emulator (X Window System),
	am, bce, km, mc5i, mir, msgr, npc, xenl,
	colors#8, cols#80, it#8, lines#24, pairs#64,
	acsc=``aaffggiijjkkllmmnnooppqqrrssttuuvvwwxxyyzz{{||}}~~,
	bel=^G, blink=\E[5m, bold=\E[1m, cbt=\E[Z, civis=\E[?25l,
	clear=\E[H\E[2J, cnorm=\E[?12l\E[?25h, cr=^M,
	csr=\E[%i%p1%d;%p2%dr, cub=\E[%p1%dD, cub1=^H,
	cud=\E[%p1%dB, cud1=^J, cuf=\E[%p1%dC, cuf1=\E[C,
	cup=\E[%i%p1%d;%p2%dH, cuu=\E[%p1%dA, cuu1=\E[A,
	cvvis=\E[?12;25h, dch=\E[%p1%dP, dch1=\E[P, dl=\E[%p1%dM,
	dl1=\E[M, ech=\E[%p1%dX, ed=\E[J, el=\E[K, el1=\E[1K,
	flash=\E[?5h$<100/>\E[?5l, home=\E[H, hpa=\E[%i%p1%dG,
	ht=^I, hts=\EH, ich=\E[%p1%d@, il=\E[%p1%dL, il1=\E[L,
	ind=^J, indn=\E[%p1%dS, invis=\E[8m,
	is2=\E[!p\E[?3;4l\E[4l\E>, kDC=\E[3;2~, kEND=\E[1;2F,
	kHOM=\E[1;2H, kIC=\E[2;2~, kLFT=\E[1;2D, kNXT=\E[6;2~,
	kPRV=\E[5;2~, kRIT=\E[1;2C, kb2=\EOE, kbs=\177, kcbt=\E[Z,
	kcub1=\EOD, kcud1=\EOB, kcuf1=\EOC, kcuu1=\EOA,
	kdch1=\E[3~, kend=\EOF, kent=\EOM, kf1=\EOP, kf10=\E[21~,
	kf11=\E[23~, kf12=\E[24~, kf13=\E[1;2P, kf14=\E[1;2Q,
	kf15=\E[1;2R, kf16=\E[1;2S, kf17=\E[15;2~, kf18=\E[17;2~,
	kf19=\E[18;2~, kf2=\EOQ, kf20=\E[19;2~, kf21=\E[20;2~,
	kf22=\E[21;2~, kf23=\E[23;2~, kf24=\E[24;2~,
	kf25=\E[1;5P, kf26=\E[1;5Q, kf27=\E[1;5R, kf28=\E[1;5S,
	kf29=\E[15;5~, kf3=\EOR, kf30=\E[17;5~, kf31=\E[18;5~,
	kf32=\E[19;5~, kf33=\E[20;5~, kf34=\E[21;5~,
	kf35=\E[23;5~, kf36=\E[24;5~, kf37=\E[1;6P, kf38=\E[1;6Q,
	kf39=\E[1;6R, kf4=\EOS, kf40=\E[1;6S, kf41=\E[15;6~,
	kf42=\E[17;6~, kf43=\E[18;6~, kf44=\E[19;6~,
	kf45=\E[20;6~, kf46=\E[21;6~, kf47=\E[23;6~,
	kf48=\E[24;6~, kf49=\E[1;3P, kf5=\E[15~, kf50=\E[1;3Q,
	kf51=\E[1;3R, kf52=\E[1;3S, kf53=\E[15;3~, kf54=\E[17;3~,
	kf55=\E[18;3~, kf56=\E[19;3~, kf57=\E[20;3~,
	kf58=\E[21;3~, kf59=\E[23;3~, kf6=\E[17~, kf60=\E[24;3~,
	kf61=\E[1;4P, kf62=\E[1;4Q, kf63=\E[1;4R, kf7=\E[18~,
	kf8=\E[19~, kf9=\E[20~, khome=\EOH, kich1=\E[2~,
	kind=\E[1;2B, kmous=\E[M, knp=\E[6~, kpp=\E[5~,
	kri=\E[1;2A, mc0=\E[i, mc4=\E[4i, mc5=\E[5i, meml=\El,
	memu=\Em, op=\E[39;49m, rc=\E8, rev=\E[7m, ri=\EM,
	rin=\E[%p1%dT, rmacs=\E(B, rmam=\E[?7l, rmcup=\E[?1049l,
	rmir=\E[4l, rmkx=\E[?1l\E>, rmm=\E[?1034l, rmso=\E[27m,
	rmul=\E[24m, rs1=\Ec, rs2=\E[!p\E[?3;4l\E[4l\E>, sc=\E7,
	setab=\E[4%p1%dm, setaf=\E[3%p1%dm,
	setb=\E[4%?%p1%{1}%=%t4%e%p1%{3}%=%t6%e%p1%{4}%=%t1%e%p1%{6}%=%t3%e%p1%d%;m,
	setf=\E[3%?%p1%{1}%=%t4%e%p1%{3}%=%t6%e%p1%{4}%=%t1%e%p1%{6}%=%t3%e%p1%d%;m,
	sgr=%?%p9%t\E(0%e\E(B%;\E[0%?%p6%t;1%;%?%p2%t;4%;%?%p1%p3%|%t;7%;%?%p4%t;5%;%?%p7%t;8%;m,
	sgr0=\E(B\E[m, smacs=\E(0, smam=\E[?7h, smcup=\E[?1049h,
	smir=\E[4h, smkx=\E[?1h\E=, smm=\E[?1034h, smso=\E[7m,
	smul=\E[4m, tbc=\E[3g, u6=\E[%i%d;%dR, u7=\E[6n,
	u8=\E[?1;2c, u9=\E[c, vpa=\E[%i%p1%dd,
  ```
## Terminfo工具tput
  linux下有tput和reset两个命令行工具可以初始化和操作一个终端。tput 命令将通过 terminfo 数据库对您的终端会话进行初始化和操作。通过使用 tput，您可以更改几项终端功能，如移动或更改光标、更改文本属性，以及清除终端屏幕的特定区域。
### 光标属性
  在shell脚本或命令行中，可以利用tput命令改变光标属性。
  ```
tput clear      # 清除屏幕
tput sc         # 记录当前光标位置
tput rc         # 恢复光标到最后保存位置
tput civis      # 光标不可见
tput cnorm      # 光标可见
tput cup x y    # 光标按设定坐标点移动
  ```
### 文本属性
  tput可使终端文本加粗、在文本下方添加下划线、更改背景颜色和前景颜色，以及逆转颜色方案等。
  ```
  tput Color Capabilities:

tput setab [0-7] – Set a background color using ANSI escape
tput setb [0-7] – Set a background color
tput setaf [0-7] – Set a foreground color using ANSI escape
tput setf [0-7] – Set a foreground color

Color Code for tput:

0 – Black
1 – Red
2 – Green
3 – Yellow
4 – Blue
5 – Magenta
6 – Cyan
7 – White

tput Text Mode Capabilities:

tput bold – Set bold mode
tput dim – turn on half-bright mode
tput smul – begin underline mode
tput rmul – exit underline mode
tput rev – Turn on reverse mode
tput smso – Enter standout mode (bold on rxvt)
tput rmso – Exit standout mode
tput sgr0 – Turn off all attributes
  ```
  
## Shell curses functions
```bash
################################################################
function addch
{
    addstr "${1:0:1}"
    return ${?}
}
################################################################
function addstr
{
    [[ "_${1}" != "_" ]] &&
      BUF_SCREEN="${BUF_SCREEN}${1}"
    return ${?}
}
################################################################
function attroff
{
    addstr "${CMD_ATTROFF}"
    return ${?}
}
################################################################
function attron
{
    return 0
}
################################################################
function attrset
{
    addstr "$( ${CMD_ATTRSET} ${1} )"
    return ${?}
}
################################################################
function beep
{
    addstr "${CMD_BEEP}"
    return ${?}
}
################################################################
function chkcols
{
    chkint ${1} ${2} &&
      (( ${2} >= 0 )) &&
      (( ${2} <= ${MAX_COLS} )) &&
      return 0

    ROW_NBR="24"
    COL_NBR="1"

    eval addstr \"${CMD_MOVE}\" &&
      clrtoeol &&
      addstr "${1}: Invalid column number" >&2 &&
      refresh &&
      ${ERROR_PAUSE} &&
      eval addstr \"${CMD_MOVE}\" &&
      clrtoeol &&
      refresh

    return 1
}
################################################################
function chkint
{
    let '${2} + 0' > ${DEV_NULL} 2>&1 &&
      return 0

    ROW_NBR="24"
    COL_NBR="1"

    eval addstr \"${CMD_MOVE}\" &&
      clrtoeol &&
      addstr "${1}: argument not a number" >&2 &&
      refresh &&
      ${ERROR_PAUSE} &&
      eval addstr \"${CMD_MOVE}\" &&
      clrtoeol &&
      refresh

    return 1
}
################################################################
function chklines
{
    chkint ${1} ${2} &&
      (( ${2} >= 0 )) &&
      (( ${2} <= ${MAX_LINES} )) &&
      return 0

    ROW_NBR="24"
    COL_NBR="1"

    eval addstr \"${CMD_MOVE}\" &&
      clrtoeol &&
      addstr "${1}: Invalid line number" >&2 &&
      refresh &&
      ${ERROR_PAUSE} &&
      eval addstr \"${CMD_MOVE}\" &&
      clrtoeol &&
      refresh

    return 1
}
################################################################
function chkparm
{
    [[ "_${2}" = "_" ]] &&
      move 24 1 &&
      clrtoeol &&
      addstr "${1}: Missing parameter" >&2 &&
      refresh &&
      ${ERROR_PAUSE} &&
      move 24 1 &&
      clrtoeol &&
      return 1

    return 0
}
################################################################
function clear
{
    addstr "${CMD_CLEAR}"
    return ${?}
}
################################################################
function clrtobol
{
    addstr "${CMD_CLRTOBOL}"
    return ${?}
}
################################################################
function clrtobot
{
    addstr "${CMD_CLRTOEOD}"
    return ${?}
}
################################################################
function clrtoeol
{
    addstr "${CMD_CLRTOEOL}"
    return ${?}
}
################################################################
function delch
{
    addstr "${CMD_DELCH}"
    return ${?}
}
################################################################
function deleteln
{
    addstr "${CMD_DELETELN}"
    return ${?}
}
################################################################
function endwin
{
    unset MAX_LINES
    unset MAX_COLS
    unset BUF_SCREEN
    return ${?}
}
################################################################
function getch
{
    IFS='' read -r -- TMP_GETCH
    STATUS="${?}"
#     ${CMD_ECHO} "${TMP_GETCH}"
    eval \${CMD_ECHO} ${OPT_ECHO} \"\${TMP_GETCH}\"
    return ${STATUS}
}
################################################################
function getstr
{
    IFS="${IFS_CR}"
    getch
    STATUS="${?}"
    IFS="${IFS_NORM}"
    return ${STATUS}
}
################################################################
function getwd
{
    getch
    return ${?}
}
################################################################
function initscr
{
    PGMNAME="Bourne Shell Curses demo"
    DEV_NULL="/dev/null"
    CMD_TPUT="tput"			# Terminal "put" command

    eval CMD_MOVE=\`echo \"`tput cup`\" \| sed \\\
-e \"s/%p1%d/\\\\\${1}/g\" \\\
-e \"s/%p2%d/\\\\\${2}/g\" \\\
-e \"s/%p1%02d/\\\\\${1}/g\" \\\
-e \"s/%p2%02d/\\\\\${2}/g\" \\\
-e \"s/%p1%03d/\\\\\${1}/g\" \\\
-e \"s/%p2%03d/\\\\\${2}/g\" \\\
-e \"s/%p1%03d/\\\\\${1}/g\" \\\
-e \"s/%d\\\;%dH/\\\\\${1}\\\;\\\\\${2}H/g\" \\\
-e \"s/%p1%c/'\\\\\\\`echo \\\\\\\${1} P | dc\\\\\\\`'/g\" \\\
-e \"s/%p2%c/'\\\\\\\`echo \\\\\\\${2} P | dc\\\\\\\`'/g\" \\\
-e \"s/%p1%\' \'%+%c/'\\\\\\\`echo \\\\\\\${1} 32 + P | dc\\\\\\\`'/g\" \\\
-e \"s/%p2%\' \'%+%c/'\\\\\\\`echo \\\\\\\${2} 32 + P | dc\\\\\\\`'/g\" \\\
-e \"s/%p1%\'@\'%+%c/'\\\\\\\`echo \\\\\\\${1} 100 + P | dc\\\\\\\`'/g\" \\\
-e \"s/%p2%\'@\'%+%c/'\\\\\\\`echo \\\\\\\${2} 100 + P | dc\\\\\\\`'/g\" \\\
-e \"s/%i//g\;s/%n//g\"\`

    CMD_CLEAR="$( ${CMD_TPUT} clear 2>${DEV_NULL} )"	  # Clear display
    CMD_LINES="$( ${CMD_TPUT} lines 2>${DEV_NULL} )"	  # Number of lines on display
    CMD_COLS="$( ${CMD_TPUT} cols 2>${DEV_NULL} )"	  # Number of columns on display
    CMD_CLRTOEOL="$( ${CMD_TPUT} el 2>${DEV_NULL} )"	  # Clear to end of line
    CMD_CLRTOBOL="$( ${CMD_TPUT} el1 2>${DEV_NULL} )"	  # Clear to beginning of line
    CMD_CLRTOEOD="$( ${CMD_TPUT} ed 2>${DEV_NULL} )"	  # Clear to end of display
    CMD_DELCH="$( ${CMD_TPUT} dch1 2>${DEV_NULL} )"	  # Delete current character
    CMD_DELETELN="$( ${CMD_TPUT} dl1 2>${DEV_NULL} )"	  # Delete current line
    CMD_INSCH="$( ${CMD_TPUT} ich1 2>${DEV_NULL} )"	  # Insert 1 character
    CMD_INSERTLN="$( ${CMD_TPUT} il1 2>${DEV_NULL} )"  # Insert 1 Line
    CMD_ATTROFF="$( ${CMD_TPUT} sgr0 2>${DEV_NULL} )"  # All Attributes OFF
    CMD_ATTRSET="${CMD_TPUT}"			  # requires arg ( rev, blink, etc )
    CMD_BEEP="$( ${CMD_TPUT} bel 2>${DEV_NULL} )"	  # ring bell
    CMD_LISTER="cat"
    CMD_SYMLNK="ln -s"
    CMD_ECHO="echo"
    CMD_ECHO="print"
    OPT_ECHO='-n --'
    CMD_MAIL="mail"
    WHOAMI="${LOGNAME}@$( uname -n )"
    WRITER="dfrench@mtxia.com"
    CMD_NOTIFY="\${CMD_ECHO} ${OPT_ECHO} \"\${PGMNAME} - \${WHOAMI} - \$( date )\" | \${CMD_MAIL} \${WRITER}"
    ERROR_PAUSE="sleep 2"
    
    case "_$( uname -s )" in
        "_Windows_NT") ${DEV_NULL}="NUL";
                       CMD_SYMLNK="cp";;
#              "_Linux") CMD_ECHO="echo -e";;
    esac
    
    IFS_CR="$'\n'"
    IFS_CR="
"
    IFS_NORM="$' \t\n'"
    IFS_NORM=" 	
"

    MAC_TIME="TIMESTAMP=\`date +\"%y:%m:%d:%H:%M:%S\"\`"
    MAX_LINES=$( ${CMD_TPUT} lines )
    MAX_COLS=$( ${CMD_TPUT} cols )
    BUF_SCREEN=""
    BUF_TOT=""

    return 0
}
################################################################
function insch
{
    addstr "${CMD_INSCH}"
    return ${?}
}
################################################################
function insertln
{
    addstr "${CMD_INSERTLN}"
    return ${?}
}
################################################################
function move
{
#     chklines "${0}" "${1}" \
#     && chkcols "${0}" "${2}" \
#
################################################################
# HEATH-KIT MOVE COMMAND
#    addstr "Y${1} ${2}"
# VT100 MOVE COMMAND
#    addstr "[${1};${2}H"
# TPUT MOVE COMMAND
    eval addstr \"${CMD_MOVE}\"
# HP TERMINAL MOVE COMMAND
#   addstr "&a${1}y${2}C"
################################################################
#  add your move command below this line

    return ${?}
}
################################################################
function mvaddch
{
    move "${1}" "${2}" &&
      addch "${3}"
    return ${?}
}
################################################################
function mvaddstr
{
    move "${1}" "${2}" &&
      addstr "${3}"
    return ${?}
}
################################################################
function mvclrtobol
{
    move "${1}" "${2}" &&
      clrtobol
    return ${?}
}
################################################################
function mvclrtobot
{
    move "${1}" "${2}" &&
      clrtobot
    return ${?}
}
################################################################
function mvclrtoeol
{
    move "${1}" "${2}" &&
      clrtoeol
    return ${?}
}
################################################################
function mvcur
{
    chklines "${0}" "${1}" &&
      chkcols "${0}" "${2}" &&
      eval \"${CMD_MOVE}\"
    return ${?}
}
################################################################
function mvdelch
{
    move "${1}" "${2}" &&
      addstr "${CMD_DELCH}"
    return ${?}
}
################################################################
function mvinsch
{
    move "${1}" "${2}" &&
      addstr "${CMD_INSCH}"
    return ${?}
}
################################################################
function refresh
{
    if [[ "_${1}" != "_" ]]
    then
        eval \${CMD_ECHO} \${OPT_ECHO} \"\${${1}}\"
    else
        ${CMD_ECHO} ${OPT_ECHO} "${BUF_SCREEN}"
        BUF_TOT="${BUF_TOT}${BUF_SCREEN}"
        BUF_SCREEN=""
    fi
    return 0
}
################################################################
function savescr
{
    [[ "_${DEV_NULL}" != "_${1}" ]] && 
      eval ${1}="\"\${BUF_TOT}\""
    BUF_TOT=""
    return ${?}
}
################################################################
```

### A sample use shell curses functions
```bash
initscr
clear

for i in `seq 1 20`;
do
    lines=`tput lines`
    r=${RANDOM:0:2}
    for j in `seq $r $lines`;
    do
        mvaddstr $j $i "#"
    done
    refresh
    sleep 1
done
move 23 1
```
