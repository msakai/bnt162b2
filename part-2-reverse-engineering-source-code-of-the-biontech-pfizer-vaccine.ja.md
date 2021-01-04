---
title: "Reverse Engineering Source Code of the Biontech Pfizer Vaccine: Part 2"
date: 2020-12-31T12:22:03+01:00
draft: false
images:
 - https://berthub.eu/articles/dna-codon-table.png
---
<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:site" content="@powerdns_bert">
<meta name="twitter:creator" content="@powerdns_bert">
<meta name="twitter:title" content="Reverse Engineering the source code of the BioNTech/Pfizer SARS-CoV-2 Vaccine: Part 2">
<meta name="twitter:description" content="In short: the vaccine mRNA has been optimized by changing bits of RNA from (say) `UUU` to `UUC`, and people would like to understand how. This challenge is quite close to what cryptologists and reverse engineering people encounter regularly. On this page, you'll find all the details you need to get cracking to reverse engineer just HOW the vaccine has been optimized.">
<meta name="twitter:image" content="https://berthub.eu/articles/dna-codon-table.png">

このページの BNT162b2 ワクチンに関するデータの出典は [世界保健機関（WHO）のドキュメント](https://mednet-communities.net/inn/db/media/docs/11889.doc) です。

> このページは、取り掛かることが出来るように共有済みではあるものの、更新されます。
> ときどき更新を確認してみてください。

要約: mRNAワクチンは、RNAの一部を例えば `UUU` から `UUC` に置き換えるなどの最適化が行われていて、人々はその背後に存在するロジックを理解したいと思っています。
このチャレンジは暗号やリバースエンジニアリングの分野で出くわす問題にそっくりです。
このページではワクチンがどのように最適化されたかリバースエンジニアリングするのに必要な詳細のすべてを説明します。

これはただの楽しいパズルと思っていましたが、最適化手法を明らかにしてそれをドキュメントすることは、世界中の研究者にとって、タンパク質やワクチンのコードを設計するために役立つので、途方もなく重要だという指摘がありました。

ですので、もしワクチン研究を手助けしたいのなら、ぜひ以降を読んでみてください！

順位表
----------------
以下は、最適化アルゴリズムの、上位参加者です（訳注：この日本語訳は最新の順位表ではない可能性が高いため注意）：

<table>
<tr><th>名前</th><th>スコア</th><th>作者</th><th>コメント</th></tr>
<tr><td><a
href="https://gist.github.com/naomiajacobs/1e9de466ead8f362394cdfd581ec74fd#gistcomment-3578742">dnachisel</a></td><td>90.9%</td><td><a
href="https://twitter.com/pvieito">Pedro José Pereira Vieito</a></td><td><a
href="https://edinburgh-genome-foundry.github.io/DnaChisel/">DNAChiselアルゴリズム</a> </td></tr>
<tr><td><a
href="https://gist.github.com/naomiajacobs/1e9de466ead8f362394cdfd581ec74fd">dnachisel</a></td><td>79.5%</td><td><a
href="https://twitter.com/naomicodes">Naomi Jacobs</a></td><td><a
href="https://edinburgh-genome-foundry.github.io/DnaChisel/">DNAChiselアルゴリズム</a> </td></tr>
<tr><td><a
href="https://gist.github.com/sanxiyn/fddd1f18074076fb47e04733e6b62865">most-frequent.py</a></td><td>78.3%</td><td><a
href="https://twitter.com/sanxiyn">Seo Sanghyeon</a></td><td>python_codon_tablesに基づくコドン頻度最適化</td></tr>
<tr><td><a
href="https://github.com/unrelatedlabs/bnt162b2/blob/master/reverse.ipynb">3rd-cg.py</a></td><td>60.8%</td><td><a
href="https://twitter.com/pkuhar">Peter Kuhar</a></td><td>3文字目がGかCならば何もしない。そうでない場合にCに置き換えて、タンパク質がマッチするなら C を、そうでないなら更に G を試す。</td></tr>
<tr><td><a
href="https://github.com/berthubert/bnt162b2/blob/master/3rd-gc.go">3rd-gc.go</a></td><td>53.1%</td><td>bert
hubert</td><td>3文字目がGかCならば何もしない。そうでない場合にCに置き換えて、タンパク質がマッチするなら G を、そうでないなら更に C を試す。</td></tr>
<tr><td>Nop</td><td>27.6%</td><td></td><td>最適化を何も行わない</td></tr>
</table>

エントリーや更新は bert@hubertnet.nl もしくは [@PowerDNS_Bert](https://twitter.com/PowerDNS_Bert) まで。


BioNTech
--------
このデータを共有してくれたBioNTechに感謝する。
また、このようなワクチンが開発できるような最先端の技術を実現するために、数十年に渡って携わってきた非常に多くの研究者・研究員の方々にも感謝します。
これは驚くべき仕事です。

あまりにも素晴らしいので、このワクチンのすべてを理解したくなり、それでワクチンのmRNAがどうなっっているかをザックリ説明する「[BioNTechとファイザーのSARS-CoV-2ワクチンのソースコードのリバースエンジニアリング](https://msakai.github.io/bnt162b2/reverse-engineering-source-code-of-the-biontech-pfizer-vaccine.ja/)」（原文：[Reverse Engineering the source code of the BioNTech/Pfizer SARS-CoV-2 Vaccine](https://berthub.eu/articles/posts/reverse-engineering-source-code-of-the-biontech-pfizer-vaccine/)）という記事を書きました。
記事の内容が面白いことは請け合いますので、以降を読み進める前にこの記事を読んでおくことをオススメします。

この記事では幾つか疑問が残っていましたが、それこそが面白く、以降で取り上げる点です。

コドン最適化
----------------------
ワクチンは SARS-CoV-2 のSプロテインを*微妙に*変更したタンパク質を生成するRNAコードを含んでいます。

しかし、ワクチンのRNAコードそれ自体は、ウイルスの元のRNAコードから*大幅に*変更されています。
これは、ワクチン製造者が自然についての理解に基づいて行った変更です。

そして、我々の理解では、この変更はワクチンの有効性を*極めて大幅に*向上しています。
この変更を理解することはきっと面白いはずで、例えばそこからモデルナのワクチンには100マイクログラム必要な一方でBioNTechのワクチンには30マイクログラムで十分な理由などが分かるかも知れません。

以下は、ウイルスとBNT162b2ワクチンのRNAコードそれぞれのSタンパク質に関する部分の初めです。
違いのある箇所は感嘆符（！）で示しています。

```
ウイルス: AUG UUU GUU UUU CUU GUU UUA UUG CCA CUA GUC UCU AGU CAG UGU GUU
ワクチン: AUG UUC GUG UUC CUG GUG CUG CUG CCU CUG GUG UCC AGC CAG UGU GUU
               !   !   !   !   ! ! ! !     !   !   !   !   !            
```

RNAは文字 A, T, G, U からなる（文字通りの）文字列です。
物理的な区切りがあるわけではないですが、3文字ごとにグループ化して分析するのが合理的です。

グループ化されたそれぞれはコドンと呼ばれ、アミノ酸に対応しています。
以下ではアミノ酸を大文字で表しており、アミノ酸の列がタンパク質です。
例えばこんな風に対応しています:

```
ウイルス: AUG UUU GUU UUU CUU GUU UUA UUG CCA CUA GUC UCU AGU CAG UGU GUU
          M   F   V   F   L   V   L   L   P   L   V   S   S   Q   C   V
ワクチン: AUG UUC GUG UUC CUG GUG CUG CUG CCU CUG GUG UCC AGC CAG UGU GUU
               !   !   !   !   ! ! ! !     !   !   !   !   !            
```

比較してみると、それぞれのコドンは異なっているものの、アミノ酸の方は同一になっていることが分かります。
コドンには 4*4*4 種類がありますが、アミノ酸は20種類しかありません。
つまり、典型的には、あるコドンに対して同じアミノ酸を符号化しているコドンが2つあります。

二番目のコドンでは `UUU` が `UUC` に変更されていますが、それによりトータルではワクチンのCが一つ増えています。
三番目のコドンでは `GUU` が `GUG` に変更されていて、同様にGが一つ増えています。

**mRNAワクチンのGとCの文字の比率を上げることで、mRNAワクチンの有効性を向上することができることが知られています**。

さて、これが全てなのだとしたら、「アルゴリズムによってGとCが増えるようにする」ということで、このページはここで終わりですが、9番目のコドンを見ると `CCA` が `CCU` に変更されています。

約4千文字のワクチン中に、この変更は何度も現れています。

Our challenge
-------------
The goal is: find an algorithm that modifies the 'wild type' RNA code into
the BNT162b2 one. Because everyone would like to understand how to turn
viral RNA into an effective vaccine. The algorithm does not need to
reproduce the _exact_ RNA code of course, but it would be super nice if it
came up with something very similar, while also being brief.

To help you, I have provided the data in a number of forms, as described on
[the GitHub page](https://github.com/berthubert/bnt162b2).

> Note that in these files the `U` mentioned above appears as a `T`. `U` and
> `T` are the RNA and DNA manifestations of the same information. 

The easiest place to start might be the
'[side-by-side.csv](https://github.com/berthubert/bnt162b2/blob/master/side-by-side.csv)'
file. This lists the original and modified version of each codon, side by
side:

```
abspos,codonOrig,codonVaccine
0,ATG,ATG
3,TTT,TTC
6,GTT,GTG
...
3813,TAC,TAC
3816,ACA,ACA
3819,TAA,TGA
```

There is also an equivalency table that shows wich codons can be
interchanged without changing the amino acid output. Please find this in
[codon-table-grouped.csv](https://github.com/berthubert/bnt162b2/blob/master/codon-table-grouped.csv).
There is also a visual version
[here](https://en.wikipedia.org/wiki/DNA_and_RNA_codon_tables#Standard_DNA_codon_table).

A sample algorithm
------------------
On the [GitHub repository](https://github.com/berthubert/bnt162b2) you can
find
[3rd-gc.gp](https://github.com/berthubert/bnt162b2/blob/master/3rd-gc.go). 

This implements a simple strategy that works like this:

 * If a virus codon already ended on G or C, copy it to the vaccine mRNA
 * If not, replace last nucleotide in codon by a G, see if the amino acid
   still matches, if so, copy to the vaccine mRNA
 * Try the same with a C
 * Otherwise copy as is

Or in `golang`:

```
// base case, don't do anything
our = vir

// don't do anything if codon ends on G or C already
if(vir[2] == 'G' || vir[2] =='C') {
	fmt.Printf("Codon ended on G or C already, not doing anything.")
} else {
	prop = vir[:2]+"G"
	fmt.Printf("Attempting G substitution, new candidate '%s'. ", prop)
	if(c2s[vir] == c2s[prop]) {
		fmt.Printf("Amino acid still the same, done!")
		our = prop
	} else {
		fmt.Printf("Oops, amino acid changed. Trying C, new candidate '%s'. ", prop)
		prop = vir[:2]+"C"
		if(c2s[vir] == c2s[prop]) {
			fmt.Printf("Amino acid still the same, done!")
			our=prop
		} 
		
	}

}
```

This achieves a rather poor 53.1% match with the BioNTech RNA vaccine, but
it is a start.

When you design your algorithm, be sure to only base your choices on the
virus RNA. Do not peak into the BioNTech RNA!

If you have achieved a score beyond 53.1% please email a link to your code
to bert@hubertnet.nl (or [@PowerDNS_Bert](https://twitter.com/PowerDNS_Bert)
and I'll put it on the leader board at the top of this page!
 

Things that will help
---------------------
As with every form of reverse engineering or cryptanalysis, it helps to
understand what we are looking at.

GC ratio
--------
We know that one goal of the 'codon optimization' is to get more `C`s and
`G`s into the vaccine version of the RNA. However, there is also a limit to
that. In DNA, which is also used to manufacture the vaccine, `G` and `C`
bind together strongly, to the point that if you put too many of these
'nucleotides' in there, the DNA will no longer be replicated efficiently. 

So some modifications may actually happen to manage *down* the GC percentage of a
stretch of DNA if it was getting too high.

I [tweeted about this](https://twitter.com/PowerDNS_Bert/status/1344036143961169920) earlier.

Codon optimization
------------------
Some codons are rare in human DNA, or in certain cells. It may be that some
codons are replaced by other ones simply because they are more frequently
used by some cells. 

I [tweeted about this](https://twitter.com/PowerDNS_Bert/status/1344400081802448897)
earlier.

RNA folding
-----------
We've been looking at codons up to here. The RNA itself however does not
know about codons, there are no markers that say where a codon begins and
ends. The first codon on a protein however is always ATG (or AUG in RNA). 

RNA curls up into a shape. This shape might help evade the immune system or
it might improve translation into amino acids. This only depends on the
sequence of RNA nucleotides and not on specific codons.

You can submit RNA sequences to [this server of the Institute for
Theoretical Chemistry at the University of
Vienna](http://rna.tbi.univie.ac.at/cgi-bin/RNAWebSuite/RNAfold.cgi) and it
will fold RNA for you.  This is a very advanced server that does meticulous
calculations.

This [Wikipedia
page](https://en.wikipedia.org/wiki/Nucleic_acid_structure_prediction)
describes how this works.

It may be that some optimizations improve folding.

I am also told that this paper by Moderna (another mRNA vaccine
manufacturer) may be relevant:
[mRNA structure regulates protein expression through changes in functional
half-life](https://www.pnas.org/content/116/48/24075).
