struct ethernet{   
     unsigned char status;          //接收状态 
     unsigned char nextpage;        //下一个页
     unsigned int  length;          //以太网长度，以字节为单位
     unsigned int  destnodeid[3];   //目的网卡地址
     unsigned int  sourcenodeid[3]; //源网卡地址
     unsigned int  protocal;        //下一层协议
     unsigned char packet[1500];    //包的内容
 };  
void ne2000init()//ne2000网卡初始化  
{      
     rtl8019as_rst();      
     reg00=0x21;   //选择页0的寄存器，网卡停止运行，因为还没有初始化。          

delay_ms(10); //延时10毫秒,确保芯片进入停止模式 //使芯片处于mon和loopback模式,跟外

部网络断开
     page(0);
     reg0a=0x00;      
     reg0b=0x00;      
     reg0c=0xE0; //monitor mode (no packet receive)      
     reg0d=0xE2; //loop back mode  //使用0x40-0x4B为网卡的发送缓冲区，共12页，刚好

可以存储2个最大的以太网包。  //使用0x4c－0x7f为网卡的接收缓冲区，共52页。          

reg01=0x4C; //Pstart  接收缓冲区范围     
     reg02=0x80; //Pstop      
     reg03=0x4C; //BNRY      
     reg04=0x40; //TPSR    发送缓冲区范围     
     reg07=0xFF;/*清除所有中断标志位*/      
     reg0f=0x00;//IMR disable all interrupt      
     reg0e=0xC8; //DCR byte dma 8位dma方式     
     page(1); //选择页1的寄存器     
     reg07=0x4D; //CURR       
     reg08=0x00; //MAR0     
     reg09=0x41; //MAR1     
     reg0a=0x00; //MAR2     
     reg0b=0x80; //MAR3     
     reg0c=0x00; //MAR4     
     reg0d=0x00; //MAR5     
     reg0e=0x00; //MAR6      
     reg0f=0x00; //MAR7
     initNIC(); //初始化MAC地址和网络相关参数 //将网卡设置成正常的模式,跟外部网络

连接     
     page(0);     
     reg0c=0xCC; //RCR      
     reg0d=0xE0; //TCR      
     reg00=0x22; //这时让芯片开始工作?     
     reg07=0xFF; //清除所有中断标志位  
}  
void send_packet(union netcard *txdnet,unsigned int length)//ne2000发包子程序  
{//发送一个数据包的命令,长度最小为60字节,最大1514字节需要发送的数据包要先存放在

txdnet缓冲区     
     unsigned char i;     
     unsigned int ii; 
     page(0);      
     if(length<60) length="60";     
     for(i=0;i<3;i++)
          txdnet->etherframe.sourcenodeid[i]=my_ethernet_address.words[i];         

 txd_buffer_select=!txd_buffer_select;     
     if(txd_buffer_select)       
        reg09=0x40 ;          //txdwrite highaddress     
     else          
     reg09=0x46 ;          //txdwrite highaddress            
     reg08=0x00;    //read page address low     
     reg0b=length>>8;          //read count high     
     reg0a=length&0xFF;        //read count low;     
     reg00=0x12;    //write dma, page0          
     for(ii=4;ii<length+4;ii++)          
        reg10=txdnet->bytes.bytebuf[ii];       
     for(i=0;i<6;i++){                   //最多重发6次         
        for(ii=0;ii<1000;ii++)          //检查txp为是否为低             
           if((reg00&0x04)==0) break;                    
        if((reg04&0x01)!=0) break;      //表示发送成功                         

reg00=0x3E;   
     }  
     if(txd_buffer_select) reg04=0x40;   //txd packet start;      
     else reg04=0x46;          //txd packet start;              
     reg06=length>>8;          //high byte counter     
     reg05=length&0xFF;        //low byte counter      
     reg00=0x3E;               //to sendpacket;   
}  
bit recv_packet(union netcard *rxdnet)//ne2000收包子程序 
{      
     unsigned char i;     
     unsigned int ii;     
     unsigned char bnry,curr;          
     page(0);
     reg07=0xFF;      
     bnry="reg03";               //bnry page have read 读页指针     
     page(1);      
     curr="reg07";               //curr writepoint 8019写页指针     
     page(0);     
     if(curr==0)          
       return 0;             //读的过程出错       
     bnry="bnry"++;      if(bnry>0x7F) bnry="0x4C";      if(bnry!=curr){           

//此时表示有新的数据包在缓冲区里   //读取一包的前18个字节:4字节的8019头部,6字节目

的地址,6字节原地址,2字节协议   //在任何操作都最好返回page0         
     page(0);           
     reg09=bnry;           //read page address high         
     reg08=0x00;           //read page address low         
     reg0b=0x00;           //read count high         
     reg0a=18;             //read count low;         
     reg00=0x0A;           //read dma         
     for(i=0;i<18;i++)              
        rxdnet->bytes.bytebuf[i]=reg10;          
     i="rxdnet-">bytes.bytebuf[3];     //将长度字段的高低字节掉转         
     rxdnet->bytes.bytebuf[3]=rxdnet->bytes.bytebuf[2];          
     rxdnet->bytes.bytebuf[2]=i;             
     rxdnet->etherframe.length=rxdnet->etherframe.length-4; //去掉4个字节的CRC//表

示读入的数据包有效          
     if(((rxdnet->bytes.bytebuf[0]&0x01)==0)||(rxdnet->bytes.bytebuf[1]>0x7F)||

(rxdnet->bytes.bytebuf[1]<0x4C)||(rxdnet->bytes.bytebuf[2]>0x06)){              //

接收状态错误,或者next_page_start错误或者长度错误,将丢弃所有数据包                  

 page(1);              
     curr="reg07";       //page1             
     page(0);          //切换回page0             
     bnry="curr-1";              
     if(bnry<0x4C) bnry="0x7F";              
     reg03=bnry;       //write to bnry                  
     return 0;
     }          
     else{//表示数据包是完好的.读取剩下的数据              
       if((rxdnet->etherframe.protocal==0x0800)||(rxdnet-

>etherframe.protocal==0x0806)){              //协议为IP或ARP才接收                 

          reg09=bnry;   //read page address high                 
          reg08=4;      //read page address low                  
          reg0b=rxdnet->etherframe.length>>8;     //read count high                

          reg0a=rxdnet->etherframe.length&0xFF;   //read count low;                

          reg00=0x0A;   //read dma                  
          for(ii=4;ii<rxdnet->etherframe.length+4;ii++)                            

      rxdnet->bytes.bytebuf[ii]=reg10;              
          }              
          bnry="rxdnet-">bytes.bytebuf[1]-1;//next page start-1             
          if(bnry<0x4C) bnry="0x7F";             
          reg03=bnry;       //write to bnry                    
          return 1;         //have new packet         
      }     
    }       
    return 0; 
}
