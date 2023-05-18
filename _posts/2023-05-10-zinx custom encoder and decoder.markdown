---
layout: post
title: Zinx 中配置和使用自定义的encoder和decoder
date: 2023-05-10 15:40:00 +0800
categories: microservice
---
# Zinx 中配置和使用自定义的encoder和decoder

zinx 自身提供了编码和解码的相关类，但是在和其他系统对接时，zinx 提供的相关类无法正确的完成工作，这时候就需要使用自定义的encoder和decoder。

## 自定义decoder

要在 zinx 中实现自定义的 decoder，需要创建一个新的结构体`MyDecoder`，并为该结构体实现 `ziface.IDecoder` 接口。

`ziface.IDecoder`的定义如下

```golang
type IDecoder interface {
	IInterceptor
	GetLengthField() *LengthField
}

// 其中 IInterceptor 的定义如下
type IInterceptor interface {
	Intercept(IChain) IcResp //拦截器的拦截处理方法(由开发者定义)
}

```

`LengthField`在zinx中定义了头部相关字段的长度，具体的细节可以参考文档：https://www.yuque.com/aceld/tsgooa/lsu25bgvqoxeg6nz  
`IInterceptor`是zinx中拦截器的接口定义，具体细节可以参考文档：https://www.yuque.com/aceld/tsgooa/zn0y6xpynrzfd1z5

```golang
type MyDecoder struct {
	Tag    uint32
	Length uint32
	Data   []byte
}

func NewMyDecoder() ziface.IDecoder {
	return &MyDecoder{}
}

// 实现 GetLengthField 方法，返回相关字段长度
func (decoder *MyDecoder) GetLengthField() *ziface.LengthField {
	return &ziface.LengthField{
		MaxFrameLength:      math.MaxUint32 + 12,
		LengthFieldOffset:   8,
		LengthFieldLength:   4,
		LengthAdjustment:    0,
		InitialBytesToStrip: 0,
	}
}

// 实现 IInterceptor 接口，此处是进行解码的入口
func (decoder *MyDecoder) Intercept(chain ziface.IChain) ziface.IcResp {

	//1. 获取zinx的IMessage
	iMessage := chain.GetIMessage()
	if iMessage == nil {
		//进入责任链下一层
		return chain.ProceedWithIMessage(iMessage, nil)
	}

	//2. 获取数据
	data := iMessage.GetData()

	//3. 读取的数据不超过包头，直接进入下一层
	if len(data) < HEADER_SIZE {
		return chain.ProceedWithIMessage(iMessage, nil)
	}

    mydata := decoder.decode(data)

	//5. 将解码后的数据重新设置到IMessage中, Zinx的Router需要MsgID来寻址
	iMessage.SetMsgID(mydata.Tag)
	iMessage.SetData(mydata.Data)
	iMessage.SetDataLen(mydata.Length)

	//6. 将解码后的数据进入下一层
	return chain.ProceedWithIMessage(iMessage, *mydata)
}

func (decoder *MyDecoder) decode(data []byte) *MyDecoder {
	dc := MyDecoder{}
	//获取T

	dc.Tag = binary.BigEndian.Uint32(data[4:8])
	//获取L
	dc.Length = binary.BigEndian.Uint32(data[8:12])
	//确定V的长度
	dc.Data = make([]byte, dc.Length)

	//5.获取V
	binary.Read(bytes.NewBuffer(data[HEADER_SIZE:HEADER_SIZE+dc.Length]), binary.BigEndian, dc.Data)
	return &dc
}
```

之后，想server注册刚刚写好的decoder

```golang
s.SetDecoder(codec.NewMyDecoder()
```

## 自定义encoder

zinx中没有Encoder相关的接口定义，要实现相关的逻辑，需要创建自定义的`DataPacket`结构体，并实现`IDataPack`接口: 

```golang
type IDataPack interface {
	GetHeadLen() uint32                //获取包头长度方法
	Pack(msg IMessage) ([]byte, error) //封包方法
	Unpack([]byte) (IMessage, error)   //拆包方法
}
```

```golang
type MyDataPack struct{}

func (dp *MyDataPack) GetHeadLen() uint32 {
	// Returns the length of the message header
	return 12
}

func (dp *MyDataPack) Pack(msg ziface.IMessage) ([]byte, error) {
	// Convert the message to your custom message struct
	// Create a buffer to hold the encoded message
	buf := bytes.NewBuffer(nil)

	// Write the message header
	binary.Write(buf, binary.BigEndian, uint32(0xffffffff))
	binary.Write(buf, binary.BigEndian, msg.GetMsgID())
	binary.Write(buf, binary.BigEndian, msg.GetDataLen())

	// Write the message data
	binary.Write(buf, binary.BigEndian, msg.GetData())

	return buf.Bytes(), nil
}

func (dp *MyDataPack) Unpack(binaryData []byte) (ziface.IMessage, error) {
	// Create a buffer to read the message header
	buf := bytes.NewReader(binaryData[:12])

	// Read the message header
	var start uint32
	var id uint32
	var dataLen uint32
	binary.Read(buf, binary.BigEndian, &start)
	binary.Read(buf, binary.BigEndian, &id)
	binary.Read(buf, binary.BigEndian, &dataLen)

	// Create a buffer to read the message data
	dataBuf := bytes.NewReader(binaryData[12:])

	// Read the message data
	data := make([]byte, dataLen)
	binary.Read(dataBuf, binary.BigEndian, &data)

	// Create a new instance of your custom message struct
	myMsg := zpack.NewMsgPackage(id, data)

	return myMsg, nil
}
```

同样，在此之后需要向zinx注册DataPacket

```golang
s.SetPacket(&codec.MyDataPack{})
```
