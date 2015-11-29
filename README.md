# nRF23L01_stm32f103ze
##Realized p2p communication between stm32f103 using  nRF24L01  2.4g
##调试方案
24L01是收发双方都需要编程的器件，这就对调试方法产生了一定的要求，如果两块一起调，那么通讯不成功，根本不知道是发的问题还是收的问题，不隐晦的说，我当时也是没理清调试思路才浪费了大半天时间看着模块干瞪眼。正确的方法应该是先调试发送方，能保证发送正确，再去调接收，这样就可以有针对性的解决问题。 
      至于怎么去调发送方，先说下发送方的工作流程： 
        
      ·配置寄存器使芯片工作于发送模式后拉高CE端至少10us 
      ·读状态寄存器STATUS 
      ·判断是否是发送完成标志位置位 
      ·清标志 
      ·清数据缓冲 
        
      发送方发送-等应答-（自动重发）-触发中断。可是这样的流程就已经把接收方给牵涉进来了，就是说一定要接收方正确收到数据并且回送应答信号之后发送方才能触发中断，结束一次完整的发送。可是这跟我们的初衷不相符，我们想单独调试发送，完全抛开接收，这样就要去配置一些参数来取消自动应答，取消自动重发，让发送方达到发出数据就算成功的目的。 
             SPI_RW_Reg(WRITE_REG + EN_AA, 0x00);           //失能通道0自动应答 
             SPI_RW_Reg(WRITE_REG + EN_RXADDR, 0x00);    //失能接收通道0 
      SPI_RW_Reg(WRITE_REG + SETUP_RETR, 0x00);      //失能自动重发 
      （注：以下贴出的寄存器描述由于中文资料上有一个错误，故贴出原版英文资料） 
        
有了以上这三个配置，发送方的流程就变成了发送-触发中断。这样就抛开了接收方，可以专心去调试发送，可是怎么样才知道发送是否成功呢，要用到另外两个寄存器，STATUS和FIFO_STATUS。 
    这样就很清晰了，我们可以通过读取STATUS的值来判断是哪个事件触发了中断，寄存器4、5、6位分别对应自动重发完成中断，数据发送完成中断，数据接收完成中断。也就是说，在之前的配置下，如果数据成功发送，那么STATUS的值应该为0x2e。这样就可以作为一个检测标准，另外一个标准可以看FIFO_STATUS寄存器，第5位的描述：发送缓冲器满标志，1为满，0为有可用空间；第4位的描述：发送缓冲器空标志，1为空，0为有数据；同样可以看到接收缓冲器的对应标志。这样在数据发送成功后，发送寄存器当然应该是空的，接收缓冲因为在之前已经失能，所以也应该是空，也就是说成功发送之后的FIFO_STATUS寄存器值应该是0x11。 
      有了这两个检测标准，我们即使不用接收方也可以确定发送方是否成功发送。当发送方调试成功之后，在程序里让它一直发送，然后我们就可以去调试接收方，思路是一样的，同样说下接收方工作流程先。 
        
      ·配置寄存器使芯片工作于接收模式后拉高CE端至少130us 
      ·读状态寄存器STATUS 
      ·判断是否是接收完成标志位置位 
      ·清标志 
      ·读取数据缓冲区的数据 
      ·清数据缓冲 
        
      然后在初始化配置寄存器的时候要和发送方保持一致，比较重要的是要失能自动应答，使能通道0接收： 
             SPI_RW_Reg(WRITE_REG + EN_AA, 0x00);           //失能通道0自动应答 
      SPI_RW_Reg(WRITE_REG + EN_RXADDR, 0x01);    //接收要使能接收通道0 
      这样就可以了，接收方就可以进入接收模式去接收数据了，这次的调试就会灵活一些，因为是接收数据，可以在接收方添加一个显示设备把数据直观的显示出来，去对照看是否正确，当然还可以使用和发送方一样的方法：观察STATUS和FIFO_STATUS的值，对照寄存器描述，接收正确时STATUS的值应该是0x40，对于FIFO_STATUS的情况就多了些，因为数据宽度的不同也会造成寄存器的值不一样，24L01最大支持32字节宽度，就是说一次通讯最多可以传输32个字节的数据，在这种情况下，接收成功读数据之前寄存器值应该为0x12，读数据之后就会变成0x11；如果数据宽度定义的小于32字节，那么接收成功读数据之前寄存器值应该为0x10，读数据之后就会变成0x11。这个看起来挺复杂，其实很清晰，大家可以试着分析下，对照数据手册分析每个位的状态就可以得到结果。 
        
     到这里对nRF24L01的调试基本上就算通了，但是要明白这些只是调试方法，最终的产品如果不加上应答和重发的话那么数据的稳定性是很难保证的，所以在基本的通讯建立之后就要把发送的配置改为：  
            
      SPI_RW_Reg(WRITE_REG + EN_AA, 0x01);             //使能接收通道0自动应答 
             SPI_RW_Reg(WRITE_REG + EN_RXADDR, 0x01);      //使能接收通道0         
             SPI_RW_Reg(WRITE_REG + SETUP_RETR, 0x1a);      //自动重发10次，间隔500us 
      接收方的配置也要更改： 
             SPI_RW_Reg(WRITE_REG + EN_AA, 0x01);           //失能通道0自动应答 
      SPI_RW_Reg(WRITE_REG + EN_RXADDR, 0x01);    //接收要使能接收通道0
