# RSA加密学习


# RSA加密过程

### 1.任意选取两个不同的大质数

$$
p,q
$$

### 2.计算`p, q`的乘积

$$
n = p * q
$$

### 3.计算`n`的欧拉函数

欧拉函数：是小于`n`的正整数中与`n`互质的数的数目
$$
φ(n)=φ(p)*φ(q)=(p-1) * (q-1)
$$
已知`n`很难求出`φ(n)`，所以求`φ(n)`只能通过`p, q`来算出

### 4.选出一个与`φ(n)`互质的整数`e`

$$
1&lt;e&lt;φ(n)
$$

整数e用做加密钥。注意：所有大于p和q的素数都可用

### 5.计算出`e`对于`φ(n)`的模反元素`d`

$$
(d*e) mod φ(n) = 1
$$

为私钥计算过程，`d`即为私钥，公式即
$$
d*e-1=kφ(n),k≥1
$$
所以，知道`e`和`φ(n)`可以轻易算出`d`

### 6.公钥与私钥

公钥：
$$
key=(e, n)
$$
私钥：
$$
key=(d,n)
$$

### 7.明文加密

`M`为明文，`e, n`为公钥，`C`为密文
$$
M^e mod n=C
$$

### 8.密文解密

`C`为密文，`d, n`为私钥，`M`为明文
$$
C^dmodn=M
$$
只根据n和e（注意：不是p和q）要计算出d是不可能的。因此，任何人都可对明文进行加密，但只有授权用户（知道d）才可对密文解密



---

> 作者: lo1see  
> URL: http://localhost:1313/posts/rsa%E5%8A%A0%E5%AF%86%E5%AD%A6%E4%B9%A0/  

