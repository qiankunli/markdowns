## 深入理解Linux网络技术

函数指针，最主要的优点是，可以根据不同的准则以及该对象扮演的角色进行初始化。也就是调用obj.func时，可能会为不同的obj调用不同的函数。

    struct obj{
        void (*func)(struct *sk)
    }
