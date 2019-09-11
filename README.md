# GPSPostQuantum

This repository implements the "Galbraith--Petit--Silva" quantum-resistant cryptosystem, [now published](https://link.springer.com/content/pdf/10.1007/s00145-019-09316-0.pdf) in the Journal of Cryptology. It uses as a dependency the number-theory library LiDIA, in [a fork of which](https://github.com/benediamond/LiDIA) I have added extended support for point-counting-related operations on curves over prime-power-order fields. As far as I know, it is the first implementation of this theoretically compelling post-quantum cryptosystem.

This is still a proof-of-concept, and not suitable (or practical, yet) for actual cryptographic use.

## Technical Notes

### Overview

The main innovation I introduce to the original work of GPS is the use of a prime _p_ == 1 (mod 8) and a curve j_0 over F_p for which the endomorphism ring O_0 = End(j_0) resides in the third row of the table of [Petit and Lauter](https://eprint.iacr.org/2017/962.pdf), Prop. 1. For such j_0, the specialized modular polynomials \Phi_l(X, j_0) lack repeated roots over F_p^2; as such, the method of _Elkies_ serves to efficiently factor E(j_0)'s division polynomials \psi_l(X). More precisely, I first re-situate all generation of torsion bases on E_0 in a pre-processing step, so that, during walking, only random combinations of _pre-calculated_ basis elements must be generated (this "hybrid" approach resembles the work of [De Feo--Jao--Plût](https://eprint.iacr.org/2011/506.pdf)). The approach of Elkies (with improvements due to Müller) efficiently yields these torsion bases, as the order-(_l_ - 1) / 2 factors of \psi_l(X) the algorithm generates correspond exactly to order-_l_ subgroups of E[_l_], and hence to candidate basis elements of E[_l_].

Thus, I never directly factor division polynomials (except when calculating E[_q_], where _q_ is as in [Petit and Lauter](https://eprint.iacr.org/2017/962.pdf), Prop. 1.).

One consequence of this approach is that prime _powers_ l^e can no longer be used, at least [not in any obvious way](http://citeseerx.ist.psu.edu/viewdoc/summary?doi=10.1.1.48.3312).

### Performance and Parallelization

Targeting 4 bits of post-quantum security, I performed this preprocessing step for a > 16-bit prime _p_ with low multiplicative order modulo those those primes _l_ comprising _p_'s mixing walk. More precisely, the prime _p_ = 577513 minimizes, over those primes _p_ > 2 ^ {4 * 4}, the maximum, over those _l_i_ required to obtain adequate mixing in the supersingular isogeny graph of _p_,  multiplicative order of -_p_ modulo {1, ..., l_i - 1} / (x ~ -x). In fact, 577513 obtains about 4.78 bits of post-quantum security.

The idea, again inspired by De Feo--Jao--Plût (see also [CSIDH](https://eprint.iacr.org/2018/383.pdf)), is that poor performance comes primarily from the high degrees of the extension fields over which the bases of the torsion subgroups E[_l_] reside. These degrees, in turn, are controlled by the the order of the restriction of the p^2-order Frobenius to E[_l_], or in other words by the multiplicative order of -_p_ (mod l). The resulting GPS object is serialized in `serializations/l = 4.txt`.

This pre-generation step still takes some time, unfortunately---not because of finding factors of each divpol, but because of finding roots of these factors (over extension fields of appropriate degree). I can't currently construct extension fields using as towers (that is, using successive irreducible polynomials); even if I could, I suppose there are _two_ polynomials for which we need simultaneous roots (namely, the two separate factors of the divpol), and that even towers wouldn't help in constructing _both_ (at least not asymptotically).

Beyond these theoretical matters, both preprocessing and signing would benefit vastly from parallelization. See [#4](https://github.com/benediamond/GPSPostQuantum/issues/4) for more discussion on this. Put simply, parallelization would impose virtually no overhead, in the sense that with multiple cores would simply divide the total time correspondingly.

As it stands, performance is frankly horrible. Signing for _p_ = 577513 takes a horrific 30 minutes or so. This time could be divided by however many parallel threads one had access to. I think also improved field arithmetic is also very necessary here.

Verifying takes about 10 seconds, and uses the [modular polynomials database](https://math.mit.edu/~drew/ClassicalModPolys.html) of Andrew Sutherland, whom I thank. The relevant files (i.e., for each l_i^{e_i}) must placed in a top-level directory `/modular/`.

### Differences from Galbraith--Petit--Silva: Detail

Here are few more miscellaneous notes.

* As mentioned above, I move torsion-basis-generation to a preprocessing step. During walking, for each l, only the coefficients of each Q_i with respect to the pre-computed l-torsion basis must be randomly generated.
* As a consequence of this, it is necessary, for the success of Algorithm 2, that the powersmooth norm S of the endormorphism ring ideal generated by Algorithm 1 be known in advance. Resultingly, it not possible to dynamically adjust the powersmooth numbers S_1 and S_2 as suggested in GPS, Algorithm 1. I therefore require that S_1 * S_2 equal the powersmooth norm S_1 * S_2 originally chosen. On the other hand, it is _not_ necessary, as appears to be implied by Algorithm 3, that these numbers _individually_ (at least at first) agree with their counterparts specified in section 4.5, but only that their product does. I therefore choose, in order to ensure that the right-hand sides of lines 11 and 18 be large enough, an "imbalanced" approach where S_2 is much larger than S_1 (_only in Algorithm 1, that is_).
* As another consequence, line 14 of Algorithm 1 can't go through as written. Instead, I keep regenerating _beta_1_ until the Legendre condition of line 14 is true immediately (that is, with r = 1). If either of the two Cornacchias fails (with a negative right-hand-side), I re-start the entire procedure using a new _N_.
* The "linear algebra modulo _N_" of line 13, described further in [Kohel, Lauter, Petit, and Tignol](https://eprint.iacr.org/2014/505.pdf), section 4.3, requires an isomorphism O/NO --> M_2(Z / NZ). A pair (x, y) satisfying -p == x^2 - q * y^2 (mod N), with the aid of which which this isomorphism can be computed (see Keith Conrad's [explainer](https://kconrad.math.uconn.edu/blurbs/ringtheory/quaternionalg.pdf) on quaterion algebras, Theorem 4.16), is constructed most naturally when -q is a square modulo N (see Theorem 4.14---but also cf. Corollary 4.24). I thus enforce in Algorithm 1, _jacobi_(-_q_, _N_) == 1.

There are more changes that I can't recall; I will try to add these as time goes on.

## Building and Usage Instructions

First, build and install [my fork of LiDIA](https://github.com/benediamond/LiDIA). Also install [Crypto++](https://www.cryptopp.com/).

To build GPS, navigate to the main repo directory and run
```bash
make
```
To execute it, run
```bash
./build/apps/main
```
The example usage given in `main.cc` imports a GPS object from a file, generates a keypair, and signs and verifies a message. Here is an example output:

```bash
embedding: 
(1 0 )
(0 1 ) mod 577513

restriction: 
(1 0 )
(0 1 ) mod 577513

l = 3, stepped: (198791 * x+ 576664)
l = 11, stepped: (472749 * x+ 456069)
l = 17, stepped: (297451 * x+ 313825)
l = 23, stepped: (166315 * x+ 114295)
l = 31, stepped: (90145 * x+ 214761)
l = 41, stepped: (65147 * x+ 81682)
l = 47, stepped: (380260 * x+ 330938)
l = 59, stepped: (12416 * x+ 226356)
l = 67, stepped: (387572 * x+ 500704)
l = 73, stepped: (220079 * x+ 223448)
l = 83, stepped: (549120 * x+ 386454)
l = 97, stepped: (163397 * x+ 284035)
```

<details><summary>Truncated...</summary>
<p>

```bash
extending the walk...
l = 5, stepped: (21693 * x+ 220341)
l = 13, stepped: (412806 * x+ 150569)
l = 19, stepped: (396866 * x+ 376009)
l = 29, stepped: (52758 * x+ 125676)
l = 37, stepped: (360803 * x+ 28199)
l = 43, stepped: (248432 * x+ 67312)
l = 53, stepped: (335304 * x+ 230578)
l = 61, stepped: (457212 * x+ 163453)
l = 71, stepped: (200275 * x+ 447401)
l = 79, stepped: (44725 * x+ 527574)
l = 89, stepped: (174688 * x+ 507992)
l = 101, stepped: (492598 * x+ 365888)
rerouting...
l = 3, stepped: (198791 * x+ 576664)
l = 11, stepped: (303439 * x+ 468780)
l = 17, stepped: (571058 * x+ 164343)
l = 23, stepped: (306881 * x+ 152179)
l = 31, stepped: (261127 * x+ 332914)
l = 41, stepped: (465724 * x+ 139136)
l = 47, stepped: (546780 * x+ 325470)
l = 59, stepped: (382897 * x+ 373148)
l = 67, stepped: (159466 * x+ 546896)
l = 73, stepped: (314499 * x+ 331453)
l = 83, stepped: (93878 * x+ 239773)
l = 97, stepped: (457773 * x+ 58857)
l = 5, stepped: (179913 * x+ 124902)
l = 13, stepped: (68994 * x+ 387992)
l = 19, stepped: (295650 * x+ 200557)
l = 29, stepped: (163445 * x+ 279505)
l = 37, stepped: (480769 * x+ 169605)
l = 43, stepped: (123072 * x+ 180081)
l = 53, stepped: (344140 * x+ 484918)
l = 61, stepped: (78870 * x+ 42884)
l = 71, stepped: (313666 * x+ 342205)
l = 79, stepped: (498 * x+ 174880)
l = 89, stepped: (346830)
l = 101, stepped: (492598 * x+ 365888)
extending the walk...
l = 5, stepped: (21693 * x+ 220341)
l = 13, stepped: (449509 * x+ 32521)
l = 19, stepped: (132616 * x+ 63211)
l = 29, stepped: (448780 * x+ 481904)
l = 37, stepped: (46316 * x+ 447876)
l = 43, stepped: (272648 * x+ 268848)
l = 53, stepped: (155445 * x+ 261244)
l = 61, stepped: (487639 * x+ 212356)
l = 71, stepped: (285630 * x+ 14854)
l = 79, stepped: (107977 * x+ 397282)
l = 89, stepped: (47913 * x+ 457388)
l = 101, stepped: (123994 * x+ 378367)
rerouting...
l = 3, stepped: (378722 * x+ 239807)
l = 11, stepped: (51715 * x+ 212481)
l = 17, stepped: (276260 * x+ 454810)
l = 23, stepped: (488652 * x+ 435430)
l = 31, stepped: (52599 * x+ 92077)
l = 41, stepped: (500776 * x+ 37016)
l = 47, stepped: (331552 * x+ 384894)
l = 59, stepped: (573354 * x+ 100189)
l = 67, stepped: (68402 * x+ 218171)
l = 73, stepped: (368547 * x+ 76232)
l = 83, stepped: (458523 * x+ 18241)
l = 97, stepped: (476606 * x+ 105688)
l = 5, stepped: (490503 * x+ 398493)
l = 13, stepped: (140779 * x+ 514265)
l = 19, stepped: (300898 * x+ 140257)
l = 29, stepped: (120211 * x+ 512347)
l = 37, stepped: (381095 * x+ 560915)
l = 43, stepped: (163142 * x+ 51719)
l = 53, stepped: (40753 * x+ 291801)
l = 61, stepped: (527590 * x+ 208844)
l = 71, stepped: (549892 * x+ 251312)
l = 79, stepped: (122096 * x+ 62935)
l = 89, stepped: (284487 * x+ 561247)
l = 101, stepped: (123994 * x+ 378367)
extending the walk...
l = 5, stepped: (267775 * x+ 219620)
l = 13, stepped: (558636 * x+ 495372)
l = 19, stepped: (388094 * x+ 548597)
l = 29, stepped: (127706 * x+ 323259)
l = 37, stepped: (384960 * x+ 37114)
l = 43, stepped: (477000 * x+ 476593)
l = 53, stepped: (208949 * x+ 243998)
l = 61, stepped: (225447 * x+ 540243)
l = 71, stepped: (13696 * x+ 440702)
l = 79, stepped: (569592 * x+ 154509)
l = 89, stepped: (25557 * x+ 320473)
l = 101, stepped: (118146 * x+ 1895)
rerouting...
l = 3, stepped: (298803 * x+ 307007)
l = 11, stepped: (474879 * x+ 326440)
l = 17, stepped: (109553 * x+ 221941)
l = 23, stepped: (114175 * x+ 35391)
l = 31, stepped: (2270 * x+ 229306)
l = 41, stepped: (383307 * x+ 392431)
l = 47, stepped: (151607 * x+ 361178)
l = 59, stepped: (390093 * x+ 16800)
l = 67, stepped: (269709 * x+ 45022)
l = 73, stepped: (397917 * x+ 496466)
l = 83, stepped: (290271 * x+ 379439)
l = 97, stepped: (546604 * x+ 183797)
l = 5, stepped: (487201 * x+ 417885)
l = 13, stepped: (26900 * x+ 433661)
l = 19, stepped: (116974 * x+ 487953)
l = 29, stepped: (557876 * x+ 417137)
l = 37, stepped: (320326 * x+ 103448)
l = 43, stepped: (406851 * x+ 436554)
l = 53, stepped: (36809 * x+ 136200)
l = 61, stepped: (305251 * x+ 244414)
l = 71, stepped: (46407 * x+ 226950)
l = 79, stepped: (220931 * x+ 49638)
l = 89, stepped: (161489 * x+ 521970)
l = 101, stepped: (118146 * x+ 1895)
extending the walk...
l = 5, stepped: (303950 * x+ 524635)
l = 13, stepped: (560355 * x+ 440795)
l = 19, stepped: (135540 * x+ 364551)
l = 29, stepped: (314793 * x+ 429770)
l = 37, stepped: (362785 * x+ 117300)
l = 43, stepped: (32606 * x+ 575706)
l = 53, stepped: (375778 * x+ 94131)
l = 61, stepped: (384438 * x+ 109924)
l = 71, stepped: (516893 * x+ 125232)
l = 79, stepped: (270755 * x+ 74932)
l = 89, stepped: (179342 * x+ 392145)
l = 101, stepped: (49769 * x+ 107437)
rerouting...
l = 3, stepped: (298803 * x+ 307007)
l = 11, stepped: (474879 * x+ 326440)
l = 17, stepped: (407921 * x+ 8030)
l = 23, stepped: (298624 * x+ 109658)
l = 31, stepped: (309275 * x+ 121696)
l = 41, stepped: (416549 * x+ 47169)
l = 47, stepped: (156379 * x+ 222040)
l = 59, stepped: (388434 * x+ 369079)
l = 67, stepped: (160979 * x+ 189389)
l = 73, stepped: (532548 * x+ 370153)
l = 83, stepped: (164214 * x+ 389870)
l = 97, stepped: (248978 * x+ 229882)
l = 5, stepped: (290779 * x+ 135368)
l = 13, stepped: (105059 * x+ 114858)
l = 19, stepped: (87911 * x+ 93108)
l = 29, stepped: (34500 * x+ 150041)
l = 37, stepped: (391298 * x+ 301476)
l = 43, stepped: (329704 * x+ 395907)
l = 53, stepped: (121504 * x+ 343144)
l = 61, stepped: (173227 * x+ 157326)
l = 71, stepped: (463701 * x+ 552468)
l = 79, stepped: (145970 * x+ 427615)
l = 89, stepped: (112363 * x+ 539509)
l = 101, stepped: (49769 * x+ 107437)
extending the walk...
l = 5, stepped: (21693 * x+ 220341)
l = 13, stepped: (449509 * x+ 32521)
l = 19, stepped: (520875 * x+ 234486)
l = 29, stepped: (105424 * x+ 560907)
l = 37, stepped: (60018 * x+ 240130)
l = 43, stepped: (77146 * x+ 342489)
l = 53, stepped: (164922 * x+ 412773)
l = 61, stepped: (126350 * x+ 497457)
l = 71, stepped: (277624 * x+ 173424)
l = 79, stepped: (314291 * x+ 174108)
l = 89, stepped: (432854 * x+ 542588)
l = 101, stepped: (510135 * x+ 381981)
rerouting...
l = 3, stepped: (198791 * x+ 576664)
l = 11, stepped: (304970 * x+ 351841)
l = 17, stepped: (392920 * x+ 248377)
l = 23, stepped: (158004 * x+ 574080)
l = 31, stepped: (298484 * x+ 36861)
l = 41, stepped: (187752 * x+ 241203)
l = 47, stepped: (89275 * x+ 525991)
l = 59, stepped: (371732 * x+ 147516)
l = 67, stepped: (361822 * x+ 411945)
l = 73, stepped: (150425 * x+ 80152)
l = 83, stepped: (32266 * x+ 145599)
l = 97, stepped: (481660 * x+ 506275)
l = 5, stepped: (194809 * x+ 466667)
l = 13, stepped: (263551 * x+ 385858)
l = 19, stepped: (452423 * x+ 132101)
l = 29, stepped: (99277 * x+ 568173)
l = 37, stepped: (388209 * x+ 122688)
l = 43, stepped: (110252 * x+ 488534)
l = 53, stepped: (83497 * x+ 526711)
l = 61, stepped: (84055 * x+ 450344)
l = 71, stepped: (462334 * x+ 54265)
l = 79, stepped: (439732 * x+ 556961)
l = 89, stepped: (369213 * x+ 447270)
l = 101, stepped: (510135 * x+ 381981)
extending the walk...
l = 5, stepped: (303950 * x+ 524635)
l = 13, stepped: (434487 * x+ 475075)
l = 19, stepped: (278936 * x+ 24568)
l = 29, stepped: (59312 * x+ 244657)
l = 37, stepped: (193075 * x+ 538943)
l = 43, stepped: (44337 * x+ 527376)
l = 53, stepped: (277367 * x+ 257724)
l = 61, stepped: (24166 * x+ 92362)
l = 71, stepped: (197554 * x+ 249041)
l = 79, stepped: (222163 * x+ 62797)
l = 89, stepped: (302504 * x+ 552697)
l = 101, stepped: (513721 * x+ 176986)
rerouting...
l = 3, stepped: (198791 * x+ 576664)
l = 11, stepped: (304970 * x+ 351841)
l = 17, stepped: (185091 * x+ 314535)
l = 23, stepped: (3125 * x+ 207018)
l = 31, stepped: (214091 * x+ 42620)
l = 41, stepped: (675 * x+ 574976)
l = 47, stepped: (306029 * x+ 110497)
l = 59, stepped: (184874 * x+ 174353)
l = 67, stepped: (76418 * x+ 501838)
l = 73, stepped: (138593 * x+ 342759)
l = 83, stepped: (491625 * x+ 46410)
l = 97, stepped: (266338 * x+ 254246)
l = 5, stepped: (234150 * x+ 44762)
l = 13, stepped: (180647 * x+ 533378)
l = 19, stepped: (135967 * x+ 299108)
l = 29, stepped: (502946 * x+ 68935)
l = 37, stepped: (85944 * x+ 365325)
l = 43, stepped: (323494 * x+ 3349)
l = 53, stepped: (237625 * x+ 558249)
l = 61, stepped: (402371 * x+ 516271)
l = 71, stepped: (40935 * x+ 120867)
l = 79, stepped: (178210 * x+ 257206)
l = 89, stepped: (61245 * x+ 370175)
l = 101, stepped: (513721 * x+ 176986)
extending the walk...
l = 5, stepped: (267775 * x+ 219620)
l = 13, stepped: (19959 * x+ 573233)
l = 19, stepped: (216687 * x+ 183920)
l = 29, stepped: (353883 * x+ 109213)
l = 37, stepped: (545207 * x+ 152392)
l = 43, stepped: (285079 * x+ 307652)
l = 53, stepped: (448972 * x+ 338462)
l = 61, stepped: (502479 * x+ 332401)
l = 71, stepped: (346515 * x+ 271770)
l = 79, stepped: (538280 * x+ 502461)
l = 89, stepped: (311905 * x+ 88587)
l = 101, stepped: (542887 * x+ 189955)
rerouting...
l = 3, stepped: (198791 * x+ 576664)
l = 11, stepped: (24379 * x+ 569059)
l = 17, stepped: (491625 * x+ 46410)
l = 23, stepped: (311333 * x+ 150118)
l = 31, stepped: (432142 * x+ 184843)
l = 41, stepped: (395761 * x+ 141150)
l = 47, stepped: (205686 * x+ 305816)
l = 59, stepped: (407148 * x+ 531212)
l = 67, stepped: (463005 * x+ 283700)
l = 73, stepped: (461375 * x+ 127805)
l = 83, stepped: (411933 * x+ 219248)
l = 97, stepped: (167996 * x+ 222500)
l = 5, stepped: (254148 * x+ 365164)
l = 13, stepped: (112106 * x+ 529639)
l = 19, stepped: (447853 * x+ 452645)
l = 29, stepped: (439975 * x+ 223769)
l = 37, stepped: (498840 * x+ 294112)
l = 43, stepped: (331787 * x+ 213988)
l = 53, stepped: (199227 * x+ 499024)
l = 61, stepped: (54938 * x+ 52554)
l = 71, stepped: (302675 * x+ 169192)
l = 79, stepped: (132645 * x+ 369085)
l = 89, stepped: (194895 * x+ 508760)
l = 101, stepped: (542887 * x+ 189955)
extending the walk...
l = 5, stepped: (267775 * x+ 219620)
l = 13, stepped: (558636 * x+ 495372)
l = 19, stepped: (242082 * x+ 409429)
l = 29, stepped: (483966 * x+ 315338)
l = 37, stepped: (188783 * x+ 77119)
l = 43, stepped: (253563 * x+ 198897)
l = 53, stepped: (148978 * x+ 180382)
l = 61, stepped: (141157 * x+ 353876)
l = 71, stepped: (543605 * x+ 571449)
l = 79, stepped: (160107 * x+ 449993)
l = 89, stepped: (246039 * x+ 133464)
l = 101, stepped: (392940 * x+ 190613)
rerouting...
l = 3, stepped: (378722 * x+ 239807)
l = 11, stepped: (274074 * x+ 106076)
l = 17, stepped: (169779 * x+ 16562)
l = 23, stepped: (408903 * x+ 356784)
l = 31, stepped: (181874 * x+ 276214)
l = 41, stepped: (514493 * x+ 432653)
l = 47, stepped: (200720 * x+ 367911)
l = 59, stepped: (232889 * x+ 160582)
l = 67, stepped: (260710 * x+ 233520)
l = 73, stepped: (295408 * x+ 210907)
l = 83, stepped: (235027 * x+ 172025)
l = 97, stepped: (276559 * x+ 289433)
l = 5, stepped: (424255 * x+ 453158)
l = 13, stepped: (428966 * x+ 247120)
l = 19, stepped: (305703 * x+ 1199)
l = 29, stepped: (343218 * x+ 107324)
l = 37, stepped: (209954 * x+ 193121)
l = 43, stepped: (201140 * x+ 5542)
l = 53, stepped: (512904 * x+ 133735)
l = 61, stepped: (475077 * x+ 463646)
l = 71, stepped: (504030 * x+ 82195)
l = 79, stepped: (298829 * x+ 311249)
l = 89, stepped: (346895 * x+ 439117)
l = 101, stepped: (392940 * x+ 190613)
extending the walk...
l = 5, stepped: (347200 * x+ 95052)
l = 13, stepped: (447124 * x+ 269166)
l = 19, stepped: (494841 * x+ 53742)
l = 29, stepped: (114586 * x+ 434955)
l = 37, stepped: (429946 * x+ 323971)
l = 43, stepped: (148103 * x+ 557997)
l = 53, stepped: (216747 * x+ 392918)
l = 61, stepped: (149926 * x+ 554466)
l = 71, stepped: (84479 * x+ 498096)
l = 79, stepped: (358062 * x+ 375255)
l = 89, stepped: (537266 * x+ 125034)
l = 101, stepped: (153542 * x+ 50689)
rerouting...
l = 3, stepped: (378722 * x+ 239807)
l = 11, stepped: (51715 * x+ 212481)
l = 17, stepped: (276260 * x+ 454810)
l = 23, stepped: (254696 * x+ 475922)
l = 31, stepped: (490203 * x+ 221962)
l = 41, stepped: (87446 * x+ 134101)
l = 47, stepped: (81518 * x+ 358712)
l = 59, stepped: (123733 * x+ 485710)
l = 67, stepped: (213654 * x+ 519089)
l = 73, stepped: (153257 * x+ 30376)
l = 83, stepped: (208883 * x+ 517445)
l = 97, stepped: (25013 * x+ 327910)
l = 5, stepped: (520416 * x+ 560468)
l = 13, stepped: (521829 * x+ 530216)
l = 19, stepped: (19768 * x+ 464514)
l = 29, stepped: (147318 * x+ 138668)
l = 37, stepped: (427021 * x+ 381286)
l = 43, stepped: (246300 * x+ 246979)
l = 53, stepped: (243335 * x+ 371438)
l = 61, stepped: (119328 * x+ 50784)
l = 71, stepped: (277293 * x+ 77447)
l = 79, stepped: (515062 * x+ 63556)
l = 89, stepped: (547479 * x+ 156361)
l = 101, stepped: (153542 * x+ 50689)
extending the walk...
l = 5, stepped: (347200 * x+ 95052)
l = 13, stepped: (503534 * x+ 198383)
l = 19, stepped: (490677 * x+ 89403)
l = 29, stepped: (548332 * x+ 103206)
l = 37, stepped: (262089 * x+ 544160)
l = 43, stepped: (320850 * x+ 447619)
l = 53, stepped: (200173 * x+ 437066)
l = 61, stepped: (465150 * x+ 93075)
l = 71, stepped: (558731 * x+ 46442)
l = 79, stepped: (52973 * x+ 351612)
l = 89, stepped: (339966 * x+ 18112)
l = 101, stepped: (101702 * x+ 132733)
rerouting...
l = 3, stepped: (378722 * x+ 239807)
l = 11, stepped: (198791 * x+ 576664)
l = 17, stepped: (28622 * x+ 511573)
l = 23, stepped: (218712 * x+ 546686)
l = 31, stepped: (492652 * x+ 329646)
l = 41, stepped: (276912 * x+ 206140)
l = 47, stepped: (281238 * x+ 125991)
l = 59, stepped: (120265 * x+ 352352)
l = 67, stepped: (387130 * x+ 163327)
l = 73, stepped: (528897 * x+ 256817)
l = 83, stepped: (423711 * x+ 392329)
l = 97, stepped: (193436 * x+ 421518)
l = 5, stepped: (482294 * x+ 484848)
l = 13, stepped: (376968 * x+ 43274)
l = 19, stepped: (296944 * x+ 54304)
l = 29, stepped: (312860 * x+ 276259)
l = 37, stepped: (421107 * x+ 181855)
l = 43, stepped: (490169 * x+ 4124)
l = 53, stepped: (169202 * x+ 120675)
l = 61, stepped: (276615 * x+ 401592)
l = 71, stepped: (123823 * x+ 213626)
l = 79, stepped: (415294 * x+ 366955)
l = 89, stepped: (356310 * x+ 565927)
l = 101, stepped: (101702 * x+ 132733)
extending the walk...
l = 5, stepped: (267775 * x+ 219620)
l = 13, stepped: (272560 * x+ 495995)
l = 19, stepped: (477785 * x+ 542954)
l = 29, stepped: (377794 * x+ 223028)
l = 37, stepped: (268858 * x+ 554875)
l = 43, stepped: (225296 * x+ 383518)
l = 53, stepped: (529718 * x+ 234959)
l = 61, stepped: (540384 * x+ 568660)
l = 71, stepped: (64648 * x+ 360548)
l = 79, stepped: (567872 * x+ 256996)
l = 89, stepped: (314577 * x+ 178984)
l = 101, stepped: (305568 * x+ 82445)
rerouting...
l = 3, stepped: (198791 * x+ 576664)
l = 11, stepped: (434863 * x+ 311834)
l = 17, stepped: (116849 * x+ 477719)
l = 23, stepped: (276863 * x+ 247006)
l = 31, stepped: (567557 * x+ 569903)
l = 41, stepped: (217455 * x+ 338500)
l = 47, stepped: (331444 * x+ 202056)
l = 59, stepped: (560210 * x+ 143020)
l = 67, stepped: (390808 * x+ 233818)
l = 73, stepped: (376907 * x+ 352461)
l = 83, stepped: (259559 * x+ 362655)
l = 97, stepped: (314348 * x+ 215023)
l = 5, stepped: (353979 * x+ 103088)
l = 13, stepped: (20826 * x+ 422255)
l = 19, stepped: (148222 * x+ 343424)
l = 29, stepped: (230998 * x+ 385368)
l = 37, stepped: (536472 * x+ 362616)
l = 43, stepped: (453402 * x+ 409662)
l = 53, stepped: (241512 * x+ 75194)
l = 61, stepped: (343803 * x+ 33927)
l = 71, stepped: (336899 * x+ 145546)
l = 79, stepped: (550222 * x+ 556975)
l = 89, stepped: (358562 * x+ 213952)
l = 101, stepped: (305568 * x+ 82445)
extending the walk...
l = 5, stepped: (21693 * x+ 220341)
l = 13, stepped: (235320 * x+ 327361)
l = 19, stepped: (97190 * x+ 46118)
l = 29, stepped: (280193 * x+ 524787)
l = 37, stepped: (193598 * x+ 568046)
l = 43, stepped: (76392 * x+ 368548)
l = 53, stepped: (190866 * x+ 227978)
l = 61, stepped: (374515 * x+ 365432)
l = 71, stepped: (311773 * x+ 235702)
l = 79, stepped: (309388 * x+ 391990)
l = 89, stepped: (29326 * x+ 350085)
l = 101, stepped: (474362 * x+ 229843)
rerouting...
l = 3, stepped: (198791 * x+ 576664)
l = 11, stepped: (207936 * x+ 59725)
l = 17, stepped: (159015 * x+ 383626)
l = 23, stepped: (216378 * x+ 552667)
l = 31, stepped: (203115 * x+ 564700)
l = 41, stepped: (506822 * x+ 353557)
l = 47, stepped: (21323 * x+ 306802)
l = 59, stepped: (32274 * x+ 49187)
l = 67, stepped: (309240 * x+ 389640)
l = 73, stepped: (397886 * x+ 265884)
l = 83, stepped: (381202 * x+ 575451)
l = 97, stepped: (114478 * x+ 159373)
l = 5, stepped: (203212 * x+ 372843)
l = 13, stepped: (497777 * x+ 122423)
l = 19, stepped: (114099 * x+ 230971)
l = 29, stepped: (526853 * x+ 25898)
l = 37, stepped: (57773 * x+ 330510)
l = 43, stepped: (341164 * x+ 175253)
l = 53, stepped: (164567 * x+ 280593)
l = 61, stepped: (84958 * x+ 144244)
l = 71, stepped: (442674 * x+ 30154)
l = 79, stepped: (254403 * x+ 418994)
l = 89, stepped: (424285 * x+ 492822)
l = 101, stepped: (474362 * x+ 229843)
```

</p>
</details>

```bash
D362 (303439 * x+ 468780) (306881 * x+ 152179) (465724 * x+ 139136) (382897 * x+ 373148) (314499 * x+ 331453) (457773 * x+ 58857) (68994 * x+ 387992) (163445 * x+ 279505) (123072 * x+ 180081) (78870 * x+ 42884) (498 * x+ 174880) (492598 * x+ 365888) (449509 * x+ 32521) (448780 * x+ 481904) (272648 * x+ 268848) (487639 * x+ 212356) (107977 * x+ 397282) (123994 * x+ 378367) (474879 * x+ 326440) (114175 * x+ 35391) (383307 * x+ 392431) (390093 * x+ 16800) (397917 * x+ 496466) (546604 * x+ 183797) (26900 * x+ 433661) (557876 * x+ 417137) (406851 * x+ 436554) (305251 * x+ 244414) (220931 * x+ 49638) (118146 * x+ 1895) (474879 * x+ 326440) (298624 * x+ 109658) (416549 * x+ 47169) (388434 * x+ 369079) (532548 * x+ 370153) (248978 * x+ 229882) (105059 * x+ 114858) (34500 * x+ 150041) (329704 * x+ 395907) (173227 * x+ 157326) (145970 * x+ 427615) (49769 * x+ 107437) (304970 * x+ 351841) (158004 * x+ 574080) (187752 * x+ 241203) (371732 * x+ 147516) (150425 * x+ 80152) (481660 * x+ 506275) (263551 * x+ 385858) (99277 * x+ 568173) (110252 * x+ 488534) (84055 * x+ 450344) (439732 * x+ 556961) (510135 * x+ 381981) (304970 * x+ 351841) (3125 * x+ 207018) (675 * x+ 574976) (184874 * x+ 174353) (138593 * x+ 342759) (266338 * x+ 254246) (180647 * x+ 533378) (502946 * x+ 68935) (323494 * x+ 3349) (402371 * x+ 516271) (178210 * x+ 257206) (513721 * x+ 176986) (19959 * x+ 573233) (353883 * x+ 109213) (285079 * x+ 307652) (502479 * x+ 332401) (538280 * x+ 502461) (542887 * x+ 189955) (558636 * x+ 495372) (483966 * x+ 315338) (253563 * x+ 198897) (141157 * x+ 353876) (160107 * x+ 449993) (392940 * x+ 190613) (447124 * x+ 269166) (114586 * x+ 434955) (148103 * x+ 557997) (149926 * x+ 554466) (358062 * x+ 375255) (153542 * x+ 50689) (198791 * x+ 576664) (218712 * x+ 546686) (276912 * x+ 206140) (120265 * x+ 352352) (528897 * x+ 256817) (193436 * x+ 421518) (376968 * x+ 43274) (312860 * x+ 276259) (490169 * x+ 4124) (276615 * x+ 401592) (415294 * x+ 366955) (101702 * x+ 132733) (434863 * x+ 311834) (276863 * x+ 247006) (217455 * x+ 338500) (560210 * x+ 143020) (376907 * x+ 352461) (314348 * x+ 215023) (20826 * x+ 422255) (230998 * x+ 385368) (453402 * x+ 409662) (343803 * x+ 33927) (550222 * x+ 556975) (305568 * x+ 82445) (235320 * x+ 327361) (280193 * x+ 524787) (76392 * x+ 368548) (374515 * x+ 365432) (309388 * x+ 391990) (474362 * x+ 229843) 46C37EA1076C44D41B766CB058CF58259AEDFDF1702E51B1931C17EBD07553 406B05B85FC02FD47C279C19874550A0D9B1A5338961B0603C5C90131925D59EDEE3FCE6B22B27181DDB92DDF9CF3A87C7169722B3A14C47BA9AAEA7C6 808505083307BC3FBE633CB379947C9820028E1D7E283BB24246B9132292B3 E4C2A18223D7E4300AE776CEA3765A9B67E6C16156D0B6681531E184DC4211 87874497441D8A0374C6F6A00DF5F58353D310FE26A4A01E7FF9E413A5793E 1E68477704AC79F61CFAF3D585DF46D0C29B31BD5B8ACD5B898BB5E83CE927 DE0765CE73FE6D894C0402B6D8611B3370D2DB1AFC64EDF6523D09D970C484B71A2488EC6C1A4A7622C9193E64067027D36A66E303A5078BAA72EB0B8D 189B540E0C09846A1113FF4C72914C9F0C3CE72F64ACB711A086822C82D4715E01FBD4A417E9FB6864ED3E2EA3AA82D06E2BB5E630AD5034A56637A59C D773B2BD879A0FD3DEFD9A15BD5FC7798F839CFB350C218A44038124B0FDAB3AB01F31991A82FF2682D68925C98BC5BB4BEE1DB8ACC467D74D34F6D31C 4182E6120D7F8302631B23EB717D6A15CC4314E68A4692CE7578D3D28CD538 DACA6B6D7900E65A5128AD9292D1DD6CC0616651D76F95B19E86615731A619 30463324A1E50FCD889942AA20226EFF5288AB2217A3BD54D39370786DB32F736C376E778EE443034E1352DBBD8516F5BE68B90113D52E2FDEF5439034 
1
```

## Contributing

I would welcome contributions by anyone interested. Feel free to raise a pull request or leave comments. See also the [issues](https://github.com/benediamond/GPSPostQuantum/issues/). Thanks for your interest!
