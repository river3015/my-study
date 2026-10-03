# 回答：xv6の組み込みコマンドと外部コマンド

## 現時点の結論

組み込みコマンドはシェルのプログラムに処理が含まれ、外部コマンドは別の実行ファイルとして読み込まれる。xv6の単純な `cd dir` はシェル自身が `chdir` を呼び、通常の外部コマンドは子プロセスで `exec` する。

以下は2026-10-03に確認したMIT公式xv6-riscvのriscvブランチに基づく。手元の `/Users/aikawamasahiro/git/xv6-riscv` のHEAD `06aad25c735fd3159bdfae5680be4eab7b1668b2` でも、今回関係する処理を読み、同じ仕組みであることを確認した。リポジトリ全体の一致を確認したわけではない。参照先は更新されうるブランチURLである。

## 仕組みと根拠

### lsの起動

`user/sh.c` のmainは、通常のコマンドに対してforkし、子側でruncmdを呼ぶ。runcmdのEXEC分岐は、入力されたコマンド名をそのままexecに渡す。このシェルにはPATHの候補ディレクトリを探索する処理がない。[公式sh.c](https://github.com/mit-pdos/xv6-riscv/blob/riscv/user/sh.c)

カーネルの名前解決は、先頭が `/` ならルートinode、それ以外なら呼び出したプロセスのcwdを出発点にする。したがってcwdが `/work` のとき `ls` は `/work/ls` を指し、`/ls` はルートの実行ファイルを指す。ユーザーの観測はこの実装で説明できるが、手元の環境で再現確認したわけではない。[公式fs.cのnamex](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/fs.c)

### cdとプロセスの状態

この文脈の「組み込み」は、シェルの実行ファイル `/sh` に処理が含まれるという意味。cd専用の実行ファイルをロードする必要がなく、起動したshのユーザーメモリに含まれる処理を実行する。カーネル起動時にロードされることを意味する言葉ではない。Cやアセンブリで書かれているかどうかも、組み込みと外部コマンドの区別にはならない。

入力した `cd dir` を認識するのはユーザー空間のsh。そのshがchdirシステムコールでカーネルに依頼し、カーネルのsys_chdirがプロセスのcwdを変更する。シェルのコマンド処理と、カーネルの提供する操作を分けて考える。

shのmainは `cd ` で始まる入力を特別扱いし、forkする前にchdirを呼ぶ。組み込みとはシェルに処理が含まれるという意味で、カーネル内部でコマンドを解釈するという意味ではない。[公式sh.c](https://github.com/mit-pdos/xv6-riscv/blob/riscv/user/sh.c)

sys_chdirはディレクトリを解決して、呼び出したプロセスのcwdを更新する。fork時は子のcwdに親のcwdのinode参照を引き継ぐ。親子は最初は同じディレクトリを参照するが、子がchdirしても親のcwdの参照は変わらない。そのため通常の外部コマンドとしてcdを実行すると、子だけが移動して終了し、対話中の親シェルは元の場所に残る。[公式sysfile.cのsys_chdir](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/sysfile.c)、[公式proc.cのkfork](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/proc.c)

### execが置き換えるもの

forkは子プロセスを作る。execは呼び出したプロセスのユーザーメモリを実行ファイルの内容に置き換える。カーネルはパスを解決し、ELFを確認してロードし、新しいページテーブル、ユーザースタック、実行開始位置を設定する。exec自体が新しいプロセスを作るわけではない。[公式exec.cのkexec](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/exec.c)

### システムコールの名前

手元のxv6のkernel/syscall.cでは、登録されたカーネル側のシステムコール処理関数はいずれもsys_で始まる。これは実装の命名規則であり、システムコールの成立条件ではない。ユーザー側ではchdir、read、writeなどの名前で呼ぶ。SYS_chdirなどの大文字の名前はシステムコール番号を表す定数。カーネルのsys_chdirはその番号に対応する処理関数であり、同名のユーザー関数ではない。なお手元のxv6では低レベルのsbrk入口だけsys_sbrkというユーザー側の名前を用い、ulib.cのsbrkがラップしているため、「ユーザー側には絶対にsys_が付かない」とも言えない。[syscall.c](/Users/aikawamasahiro/git/xv6-riscv/kernel/syscall.c:109)、[usys.pl](/Users/aikawamasahiro/git/xv6-riscv/user/usys.pl:13)

### テキスト領域と組み込みの設計判断

手元のuser/user.ldは機械語のコードを.text、定数を.rodata、初期値のある変数を.data、ゼロ初期化の変数を.bssに配置する。実行時にロードするのはCのソース全文ではなく、コンパイル・リンクしたELFのロード対象セグメント。kexecがそのセグメントを読み込む。cdを認識してchdirを呼ぶ命令も、shのコードとして実行される。[user.ld](/Users/aikawamasahiro/git/xv6-riscv/user/user.ld:6)、[exec.c](/Users/aikawamasahiro/git/xv6-riscv/kernel/exec.c:58)

cdで実行ファイルの探索が不要なのは、shが入力を組み込みとして判定し、内部の処理へ分岐するため。メモリに載っているだけで、名前から自動的に呼び出せるわけではない。

組み込みの重要な判断基準は、対話中のシェル自身の状態を変更する必要があるか。通常のfork/execによる外部コマンドは子プロセスとして動くため、子のchdirで親のcwdは変更できない。cdはシェル自身でchdirする必要がある。一方、一覧表示という独立した仕事は子プロセスのlsで実行できる。lsを組み込みとして実装することも可能だが、cwd変更のような必要性はない。

ソースファイルを分けることと、外部コマンドにすることは別の判断。cdの処理を別のCファイルに移しても、そのオブジェクトをshへリンクしてシェル内で呼び出せば組み込みのまま。別の実行ファイルを作り、それを子プロセスでexecすれば通常の外部コマンドになる。xv6では簡潔な実装としてsh.c内にcdの処理が書かれている。

## 具体例・検証

### sys_chdirを最初に読むときの見方

初回はbegin_op、end_op、ilock、iunlockの内部実装を保留してよい。呼び出しの役割を押さえた上で、struct procのcwdとstruct inodeのtypeを先に読む。cwdはcurrent working directoryの略。ipはこのコードではstruct inode *型なのでinodeを指すポインタと分かる。inode pointerを連想する慣用的な名前だが、変数名の由来について作者の説明を確認したわけではない。pはstruct proc *型で、myproc()が現在実行中のプロセスの管理情報を返す。

argstrの実装はkernel/syscall.cにあり、システムコール引数からユーザー側の文字列をカーネルのバッファへ取得する。argstr(0, path, MAXPATH)の0は最初の引数を指す。nameiはkernel/fs.cにあり、パスからinodeへの参照を取得する。単なる真偽の判定ではなく、後で解放またはcwdとして保持する参照を返す。初回はこれらの入口と戻り値を見る程度にし、ページテーブルやディスクのブロック構造まで同時に広げない。

まず本筋を「引数のパスを取得 → 移動先を探す → ディレクトリか確認 → cwdを更新」として読む。その後に、安全管理を3つに分けて見る。全関数の実装へ一度にジャンプしない。

| 操作 | 最初に押さえる役割 |
|---|---|
| begin_op / end_op | ファイルシステムのログによるトランザクション管理への参加・終了。一般のシステムコール全部に必要な処理ではない |
| ilock / iunlock | 対象inodeの情報へのアクセスを排他制御。ilockは必要ならディスクから情報も読み込む |
| iput | inodeを使っているという参照を1つ手放す。アンロックとは別の操作 |
| iunlockput | iunlockしてからiputする後始末 |

inodeはファイル・ディレクトリの種類やサイズなどを表す管理情報。ここでipは移動先のinodeへの参照である。成功経路では新しいipはp->cwdとして保持するのでiputしない。古いp->cwdへの参照はiputで手放す。移動先がディレクトリでない場合は、新しいipを保持しないためiunlockputして失敗する。

begin_opとend_op、ilockとiunlockは、字面の対応だけでなく成功・失敗それぞれの経路で対応を追う。sys_chdirでは3つのreturn経路すべてにend_opがある。ディレクトリ判定失敗時のロック解除はiunlockputに含まれる。

chdirは通常cwdというプロセスの情報を変えるだけだが、iputは最後の参照がなくなったリンク削除済みinodeのディスク領域を解放する場合がある。このためトランザクション内で呼ぶ必要がある。end_opごとに必ず独立したコミットが起きるのではなく、同時進行中の操作がなくなったときにコミットする。[log.c](/Users/aikawamasahiro/git/xv6-riscv/kernel/log.c:129)、[fs.cのilock](/Users/aikawamasahiro/git/xv6-riscv/kernel/fs.c:295)、[fs.cのiput](/Users/aikawamasahiro/git/xv6-riscv/kernel/fs.c:350)

### シェルの状態を保持する場所

シェルの一つの実行個体はプロセスとして動く。その状態は、ユーザー空間でシェルが管理するデータと、カーネルがそのプロセスについて管理するデータに分かれる。

| 管理する場所 | 例 | 変更する操作の例 |
|---|---|---|
| シェルのユーザーメモリ | シェル変数とexport属性、関数、エイリアス、オプション、履歴、ジョブの管理情報 | 代入、export、alias、set、shoptなど |
| カーネルのプロセス管理情報 | cwd、開いているファイルの一覧、umask、資源制限、シグナルの設定 | chdir、open/close/dup、umask、資源制限・シグナル設定のシステムコール |

表はBashなど一般的なUnixシェルとOSの例。すべてがxv6にあるわけではない。[Bash公式の実行環境の説明](https://www.gnu.org/software/bash/manual/html_node/Command-Execution-Environment.html)

通常のシェル変数はシェル内部のデータ。export属性を付けた変数などから、外部プログラムへ渡す環境の名前・値の一覧を作る。環境変数はカーネル全体に共有された設定ではなく、プログラムごとのユーザー空間のデータとして扱われる。環境を受け取った子が変更しても、親の変数は更新されない。

手元のxv6のstruct procにはcwdとofileがあり、シェルもほかのプロセスと同様にこの情報を持つ。cwdは文字列ではなくinodeへの参照、ofileはファイル記述子番号を添字にした開いているファイルの参照の配列。シェル変数、export、PATH探索などは今回のxv6のshには実装されていない。execのインターフェースもpathとargvの2引数で、環境一覧を渡す引数はない。[proc.h](/Users/aikawamasahiro/git/xv6-riscv/kernel/proc.h:102)、[user.h](/Users/aikawamasahiro/git/xv6-riscv/user/user.h:16)

プロセス全体にはさらにページテーブルや保存されたレジスタなどもあるが、コマンドの組み込み判断で述べた「状態」は、特に後続の入力やコマンドの動作に影響する設定・データを指している。

以下はxv6内での未実行の確認手順。ルートに `/ls` と `/mkdir` があり、`/study-demo` がまだ存在しない前提。

```sh
mkdir study-demo
cd study-demo
ls
/ls
/ls /
cd /
```

予測：`ls` は移動先に実行ファイルがないため失敗する。`/ls` は起動でき、現在のディレクトリを表示する。`/ls /` は起動する実行ファイルと表示対象をどちらも明示し、ルートの内容を表示する。手元のuser/ls.cのmainで、引数なしなら `ls(".")` を呼ぶことを確認した。

## 壁打ちで整理したこと

ユーザーはshのcd分岐を読み、入力の先頭文字で判定し、runcmdやforkを経由せず親シェルが直接chdirを呼ぶことを説明できた。次の疑問は、定義ジャンプでuser.hに到達してもchdirの本体が分からないこと。user.hは宣言であり、ユーザー側の入口はusys.plが生成するusys.Sの短いアセンブリ、カーネル側の処理本体はsysfile.cのsys_chdirであると整理した。手元には生成済みのusys.Sがあり、chdirの入口を確認できた。システムコール番号による間接呼び出しのため、通常のC関数の定義ジャンプだけで両者を追えるとは限らない。

シェルとカーネルの境界を再整理した。xv6のシェルは `/sh` というユーザープログラムであり、カーネルとは別にユーザー空間で動く。ファイルの配置パスがユーザー空間・カーネル空間を決めるわけではない。入力を解釈する処理はshが行い、chdirなどのシステムコールによる操作はカーネルが行う。システムコールでは、同じプロセスの実行がカーネル側の処理へ移り、通常は処理後にユーザー側へ戻る。カーネルという別の子プロセスを起動するわけではない。

ユーザーが説明の難しさを表明したため、まず「ユーザー空間のsh → システムコール → カーネルの処理 → shへ戻る」という関係に絞る。レジスタやトラップの詳細はこの境界が整理できてから扱う。

最初の疑問と観測を記録し、公式と手元のシェル・カーネル実装から回答を整理した。理解を確認できた範囲は上記の本人の説明に限り、ほかの説明をすべて理解済みとは扱わない。

追加の疑問に対し、組み込み先はシェルであることと、chdirの状態変更はカーネルが担うことを説明した。手元のMakefileではuser/_lsをfs.imgの入力に含め、mkfs/mkfs.cでは先頭のuser/と_を除いてルートディレクトリに登録するため、通常のビルドで `/ls` が配置されることをソースから確認した。実際に起動中のfs.imgの内容は未確認。

ユーザーは、外部コマンドを子プロセスで実行し、子でcdしても親シェルのcwdは元のままなので目的を達成できない、と自分の言葉で説明して納得を表明した。この理由について理解を確認できた。

補足：fork直後の子はシェルのコードを実行しているが、exec成功後は同じ子プロセスが外部コマンドのプログラムを実行する。子の中でシェルが外部コマンドを関数として呼び、その後シェルへ戻るわけではない。親は通常waitで子の終了を待ち、次の入力を受け付ける。また外部コマンドもファイル変更などの副作用を持てる。「副作用があるか」ではなく、「親シェル自身の状態変更が必要か」が今回の区別。

## 未解決事項・次に調べること

今回の区切りでは、ユーザーはshのcd分岐を読み、sys_chdirの場所も確認した。続ける場合は、struct procのcwdとstruct inodeのtypeを見て、sys_chdirの成功経路を追う。namei、inodeのロック、参照カウントの詳細は保留している。

その後、user/usys.plのentryとentry("chdir")、kernel/syscall.cのsyscallsとsyscallを読み、システムコールの入口をつなぐ。usys.plはアセンブリを生成するスクリプトで、シェル実行中にPerlが動くわけではない。ecallの前後のトラップ処理はさらに次の段階とする。

- `/ls` の動作と、実行ファイルのパスと表示対象の違い。
- RISC-Vのecall、トラップ、syscallディスパッチを通ってchdirやexecに到達する流れ。
- シェルがforkせずexecした場合、対話を続けられるか。
- Quiz.mdに復習問題を作成した。出題による理解確認は未実施。

## 手元のソース参照

### xv6で出てくるファイルの拡張子

| 拡張子 | 役割 | 例 |
|---|---|---|
| .c | C言語のソース。関数の処理本体などを書く | user/sh.c、kernel/sysfile.c |
| .h | Cからincludeするヘッダ。関数宣言、型、定数などを書く | user/user.h、kernel/proc.h |
| .S | Cプリプロセッサを通すアセンブリソース。命令やラベルを書く | user/usys.S、kernel/trampoline.S |
| .s | 通常、プリプロセッサを通さないアセンブリソース | .Sとの一般的な違い。今回の読解対象ではない |
| .pl | Perlスクリプト。xv6ではビルド時にアセンブリを生成 | user/usys.pl |
| .o | ソースをコンパイル／アセンブルしたオブジェクト。リンク前の機械語や再配置情報など | user/sh.o、user/usys.o |
| .ld | リンカスクリプト。セクションの配置やアドレスなどを指定 | user/user.ld、kernel/kernel.ld |
| .asm | xv6のビルドでは、objdumpで生成した逆アセンブルのテキスト | user/sh.asm |
| .sym | xv6のビルドでは、objdumpから生成したシンボル一覧のテキスト | user/sh.sym |
| .img | xv6ではファイルシステムのディスクイメージ | fs.img |
| 拡張子なし | 名前だけでは形式を特定できない。実行ファイルも含まれる | user/_sh、user/_ls、kernel/kernelはELF。Makefileはビルド手順のテキスト |

拡張子は役割を示す慣習で、ファイルの実際の形式やユーザー／カーネル空間を決めるものではない。.hにも定義を書く場合があり、宣言専用という制約はない。.Sはソースであって機械語のバイナリではない。

今回のビルドは `usys.pl → usys.S → usys.o` と進み、sh.oなどとリンクしてuser/_shを作る。mkfsはそれをディスクイメージ内の `/sh` として配置する。プログラム実行時に使うのはこのELFで、CやPerlのソースではない。[Makefile](/Users/aikawamasahiro/git/xv6-riscv/Makefile:104)

- [sh.c：外部コマンドのexec](/Users/aikawamasahiro/git/xv6-riscv/user/sh.c:79)
- [sh.c：cdの特別扱い](/Users/aikawamasahiro/git/xv6-riscv/user/sh.c:165)
- [proc.c：子へのcwdの継承](/Users/aikawamasahiro/git/xv6-riscv/kernel/proc.c:288)
- [sysfile.c：cwdの更新](/Users/aikawamasahiro/git/xv6-riscv/kernel/sysfile.c:454)
- [fs.c：相対パスの起点](/Users/aikawamasahiro/git/xv6-riscv/kernel/fs.c:699)
- [ls.c：引数なし時の対象](/Users/aikawamasahiro/git/xv6-riscv/user/ls.c:82)
