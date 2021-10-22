# RFC4664 - Framework of L2VPN
##概要
这篇RFC主要是讲解了L2VPN的主要的模块，关于L2VPN的requirement在[RFC4665](https://datatracker.ietf.org/doc/html/rfc4665)
## 主要部分
* L2VPN的概念和抽象模型
* data plane
* control plane
* management
* security

## L2VPN的概念和抽象
* point-to-point -- VPWS
* multipoint-to-multipoint -- VPLS
从下面的图大概能明白VPWS和VPLS是什么
![VPWS reference model](https://github.com/lpfwd/MyReadingNotes/blob/main/pics/VPLS_ref_model1.png?raw=true)
![VPLS reference model 1](https://github.com/lpfwd/MyReadingNotes/blob/main/pics/VPLS_ref_model1.png?raw=true)
![VPLS reference model 2](https://github.com/lpfwd/MyReadingNotes/blob/main/pics/VPLS_ref_model2.png?raw=true)
其中的概念包括：
* CE: Customer Edge
* PE: Provider Edge
其中对于VPLS，整个网络看起来就像是一台交换机，CE和PE中间是一个虚拟的swtich interface.



