# RingBuffer详解

就如RingBuffer的意思来看，它是一个环形的缓存区。实际上是一个长度为2^n次方的数组。

</br>
<img src="https://raw.githubusercontent.com/MuggleLee/PicGo/master/Disruptor/RingBuffer-structure.png" alt="RingBuffer-Structure"  width="250" height="250"  align="left">

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1.数组在内存中是占据连续的内存空间，所以在cpu缓存的时候会将在内存中读取到的内存地址后面的一部分数据也会缓存进去，从而达到性能的提升；但是链表在内存中是不连续的存储，所以只能缓存当前的内存地址。因此使用数组在cpu缓存机制中更加适合。</br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2.数组的长度是2^n，可以通过位运算能够更快的定位到元素，而且数组的下标是递增的long类型，所以不会存在index溢出。
</br>
</br>
</br>
</br>
</br>
</br>

图中每个格代表数组中的元素，数字代表序号，这个序号(Sequence)指向数组中下一个可用的元素。RingBuffer中的序号是递增的。

Disruptor对RingBuffer的访问控制策略才是真正的关键点所在。





参考资料：http://ifeve.com/disruptor/