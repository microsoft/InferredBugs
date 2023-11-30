using socket.core.Common;
using System;
using System.Collections.Generic;
using System.Net;
using System.Net.Sockets;
using System.Text;
using System.Threading;

namespace socket.core.Client
{
    /// <summary>
    /// 客户端基础类
    /// </summary>
    internal class TcpClients
    {
        /// <summary>
        /// 套接字
        /// </summary>
        private Socket socket;
        /// <summary>
        /// 发送端SocketAsyncEventArgs对象重用池，发送套接字操作
        /// </summary>
        private SocketAsyncEventArgsPool m_sendPool;
        /// <summary>
        /// 用于每个套接字I/O操作的缓冲区大小
        /// </summary>
        private int m_receiveBufferSize;
        /// <summary>
        /// 接受缓存
        /// </summary>
        private byte[] buffer_receive;       
        /// <summary>
        /// 接收对象
        /// </summary>
        private SocketAsyncEventArgs receiveSocketAsyncEventArgs;
        /// <summary>
        /// 发送时的最大的并发数
        /// </summary>
        private int m_minSendSocketAsyncEventArgs = 200;
        /// <summary>
        /// 是否连接成功
        /// </summary>
        public bool m_connect { get; set; }
        /// <summary>
        /// 连接成功事件
        /// </summary>
        internal event Action<bool> OnAccept;
        /// <summary>
        /// 接收通知事件
        /// </summary>
        internal event Action<byte[]> OnReceive;
        /// <summary>
        /// 断开连接通知事件
        /// </summary>
        internal event Action OnClose;      
       
        /// <summary>
        /// 设置基本配置
        /// </summary>
        /// <param name="receiveBufferSize">用于每个套接字I/O操作的缓冲区大小(接收端)</param>
        internal TcpClients(int receiveBufferSize)
        {
            m_receiveBufferSize = receiveBufferSize;
            m_sendPool = new SocketAsyncEventArgsPool(m_minSendSocketAsyncEventArgs);
            Init();
            m_connect = false;
        }

        /// <summary>
        /// 初始化
        /// </summary>
        private void Init()
        {
            buffer_receive = new byte[m_receiveBufferSize];
            //分配的发送对象池socketasynceventargs，不分配缓存
            SocketAsyncEventArgs saea_send;
            for (int i = 0; i < m_minSendSocketAsyncEventArgs; i++)
            {
                //预先发送端分配一组可重用的消息
                saea_send = new SocketAsyncEventArgs();
                saea_send.Completed += new EventHandler<SocketAsyncEventArgs>(ProcessSend);
                saea_send.UserToken = new AsyncUserToken();
                m_sendPool.Push(saea_send);
            }
        }

        /// <summary>
        /// 连接服务器
        /// </summary>
        /// <param name="ip">ip地址或域名</param>
        /// <param name="port">连接端口</param>
        internal void Connect(string ip, int port)
        {
            IPAddress ipaddr;
            if (!IPAddress.TryParse(ip, out ipaddr))
            {
                IPAddress[] iplist = Dns.GetHostAddresses(ip);
                if (iplist != null && iplist.Length > 0)
                {
                    ipaddr = iplist[0];
                }
            }
            IPEndPoint localEndPoint = new IPEndPoint(ipaddr, port);
            socket = new Socket(localEndPoint.AddressFamily, SocketType.Stream, ProtocolType.Tcp);
            SocketAsyncEventArgs connSocketAsyncEventArgs = new SocketAsyncEventArgs();
            connSocketAsyncEventArgs.RemoteEndPoint = localEndPoint;
            connSocketAsyncEventArgs.Completed += ProcessAccept;
            if (!socket.ConnectAsync(connSocketAsyncEventArgs))
            {
                ProcessAccept(null, connSocketAsyncEventArgs);
            }
        }

        /// <summary>
        /// 连接回调事件
        /// </summary>
        /// <param name="sender"></param>
        /// <param name="e">操作对象</param>
        private void ProcessAccept(object sender, SocketAsyncEventArgs e)
        {
            if (e.SocketError == SocketError.Success)
            {
                m_connect = true;          
                receiveSocketAsyncEventArgs = new SocketAsyncEventArgs();
                receiveSocketAsyncEventArgs.SetBuffer(buffer_receive, 0, buffer_receive.Length);
                receiveSocketAsyncEventArgs.Completed += ProcessReceive;
                socket.ReceiveAsync(receiveSocketAsyncEventArgs);
                if (OnAccept != null)
                {
                    OnAccept.BeginInvoke(true, null, null);
                }
            }
            else
            {
                if (OnAccept != null)
                {
                    OnAccept.BeginInvoke(false, null, null);
                }
            }
        }

        /// <summary>
        /// 发送数据到服务端
        /// </summary>
        /// <param name="data">数据</param>
        /// <param name="offset">偏移位</param>
        /// <param name="length">长度</param>
        internal void Send(byte[] data, int offset, int length)
        {
            if (!m_connect)
            {
                return;
            }
            SocketAsyncEventArgs sendEventArgs = m_sendPool.Pop();
            sendEventArgs.SetBuffer(data, offset, length);
            if (!socket.SendAsync(sendEventArgs))
            {
                ProcessSend(null, sendEventArgs);
            }
        }

        /// <summary>
        /// 发送回调
        /// </summary>
        /// <param name="sender"></param>
        /// <param name="e">操作对象</param>
        private void ProcessSend(object sender, SocketAsyncEventArgs e)
        {
            if (e.SocketError == SocketError.Success)
            {
                m_sendPool.Push(e);
            }
            else
            {
                CloseClientSocket(e);
            }
        }

        /// <summary>
        /// 接受回调
        /// </summary>
        /// <param name="sender"></param>
        /// <param name="e">操作对象</param>
        private void ProcessReceive(object sender, SocketAsyncEventArgs e)
        {
            if (e.BytesTransferred > 0 && e.SocketError == SocketError.Success)
            {
                byte[] data = new byte[e.BytesTransferred];
                Array.Copy(e.Buffer, e.Offset, data, 0, e.BytesTransferred);
                if (OnReceive != null)
                {
                    OnReceive.BeginInvoke(data,null,null);
                }
                //将收到的数据回显给客户端             
                if (!socket.ReceiveAsync(e))
                {
                    ProcessReceive(null, e);
                }
            }
            else
            {
                CloseClientSocket(e);
            }
        }

        /// <summary>
        /// 客户端断开一个连接
        /// </summary>
        /// <param name="e">操作对象</param>
        private void CloseClientSocket(SocketAsyncEventArgs e)
        {
            if (socket.Connected)
            {
                return;
            }
            // 关闭与客户端关联的套接字
            try
            {
                socket.Shutdown(SocketShutdown.Both);
            }
            // 抛出客户端进程已经关闭
            catch (Exception) { }
            socket.Close();
            if (OnClose != null)
            {
                OnClose.BeginInvoke(null, null);
            }
        }

        /// <summary>
        /// 客户端主动关闭连接
        /// </summary>
        internal void Close()
        {
            CloseClientSocket(receiveSocketAsyncEventArgs);
        }

    }
}
