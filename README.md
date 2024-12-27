# File
using ClassLibraryNetwork;
using System.Text.Json;
using System.Text;

namespace ConsoleClient1
{
    internal class request
    {
        public string name { get; set; }
        public string text { get; set; }

        public byte[] ToBytes()
        {
            var json = JsonSerializer.Serialize(this);
            return Encoding.UTF8.GetBytes(json);
        }

        static public clnMessage FromBytes(byte[] source)
        {
            var json = Encoding.UTF8.GetString(source, 0, source.Length);
            return JsonSerializer.Deserialize<clnMessage>(json);
        }
    }

    public class response
    {
        public byte[] file { get; set; }
        public bool success { get; set; }
        public string message { get; set; }

        public byte[] ToBytes()
        {
            var json = JsonSerializer.Serialize(this);
            return Encoding.UTF8.GetBytes(json);
        }

        static public response FromBytes(byte[] source)
        {
            var json = Encoding.UTF8.GetString(source, 0, source.Length);
            return JsonSerializer.Deserialize<response>(json);
        }
    }
}
using ClassLibraryNetwork;
using System.IO;
using System.Net.Sockets;
using System.Text;
using System.Text.Json;
using System.Xml;

namespace ConsoleClient1
{
    internal class Program
    {

        static bool quit = false;

        static async Task readConsole(NetworkStream stream)
        {
            //while (!quit)
            //{
            //    Console.Write("Enter text: ");
            //    var str = Console.ReadLine();
            //    if (str == "/q") { str = "[end]"; quit = true; }
            //    //var obj = new ClassLibraryNetwork.clnMessage() { message = str };
            //    var req = new request { name = "12345" };
            //    var bytes = req.ToBytes();
            //    stream.Write(bytes, 0, bytes.Length); // отправляем запрос
            //    await Task.Delay(100);
            //}

            //Console.Write("Enter text: ");
            var str = Console.ReadLine();
            var req = new request { name = "12345" };
            var bytes = req.ToBytes();
            stream.Write(bytes, 0, bytes.Length); // отправляем запрос
            await Task.Delay(100);
        }

        static async Task Main(string[] args)
        {

            var bClient = new locateServerClient();
            var endpoint = bClient.locateServer();
            if (endpoint == null)
            {
                Console.ForegroundColor = ConsoleColor.Red;
                Console.WriteLine("No servers found, exiting...");
                Console.ResetColor();   
                return;
            }

            TcpClient tcpClient = new TcpClient();
            tcpClient.Connect(endpoint);
            Console.WriteLine("Connected to server");
            using (var stream = tcpClient.GetStream())
            {
                Task.Run(async () => readConsole(stream)); // создаем поток для чтения сведений с консоли
                while (!quit)
                {
                    byte[] buf = new byte[512];
                    List<byte> bytes = new List<byte>();
                    while (stream.DataAvailable) // проверяем есть ли что-то в потоке
                    { 
                        int c = stream.Read(buf, 0, buf.Length); // читаем поток
                        bytes.AddRange(buf.Take(c));
                    }
                    if (bytes.Count > 0) // если что-то пришло
                    {
                        var str = Encoding.UTF8.GetString(bytes.ToArray()); // преобразуем в строку
                        //var res = new response();
                        var file = response.FromBytes(bytes.ToArray());

                        List<byte> data = new List<byte>();
                        for (int i = 4; i < file.file.Length; i += 4)
                        {
                            if (file.file[i] == 0) break;
                            data.Add(file.file[i]);
                        }
                        var text = Encoding.UTF8.GetString(data.ToArray());
                        Console.WriteLine(text);
                        Console.WriteLine(file.success);
                        Console.WriteLine(file.message);

                        var req = new request { name = "12345", text = text };

                        var byt = req.ToBytes();
                        stream.Write(byt, 0, byt.Length); // отправляем запрос
                        await Task.Delay(100);

                        //Console.WriteLine(file.file);
                        //Console.WriteLine(str);

                    }
                    await Task.Delay(300);
                }
            }
            tcpClient.Close();
        }
    }
}
using System;
using System.Collections.Generic;
using System.ComponentModel.DataAnnotations;
using System.Linq;
using System.Net;
using System.Net.NetworkInformation;
using System.Net.Sockets;
using System.Text;
using System.Threading.Tasks;

namespace ConsoleClient1
{
    internal class locateServerClient
    {
        Int16 port = -1;

        /// <summary>
        /// Найти сервер
        /// </summary>
        /// <returns></returns>
        internal IPEndPoint locateServer()

        {
            foreach (var address in Dns.GetHostAddresses(Dns.GetHostName()))
                if (address.AddressFamily == AddressFamily.InterNetwork)
                {
                    var udpClient = new UdpClient() { EnableBroadcast = true };
                    udpClient.Client.Bind(new IPEndPoint(address, 0));
                    var serverEp = new IPEndPoint(IPAddress.Any, 0);
                    int bPort = 8083;
                    Console.WriteLine($"Sending broadcast to {bPort} port over {address} network...");
                    udpClient.Send(new byte[] { 55 }, 1, new IPEndPoint(IPAddress.Broadcast, bPort));
                    try
                    {
                        udpClient.Client.ReceiveTimeout = 2000; // 2 seconds
                        var data = udpClient.Receive(ref serverEp);
                        udpClient.Close();
                        Console.WriteLine($"Server located at {serverEp.Address}:{BitConverter.ToInt16(data.Take(2).ToArray())}");
                        return new IPEndPoint(serverEp.Address, BitConverter.ToInt16(data.Take(2).ToArray()));
                    }
                    catch { }
                }
            return null;
        }




        private void DataReceived(IAsyncResult ar)
        {
            UdpClient c = (UdpClient)ar.AsyncState;
            IPEndPoint receivedIpEndPoint = new IPEndPoint(IPAddress.Any, 0);
            Byte[] receivedBytes = c.EndReceive(ar, ref receivedIpEndPoint);
            Console.WriteLine(DateTime.Now.ToString("HH:mm:ss.ff tt") + " (" + receivedBytes.Length + " bytes)");
            port = 1;
            c.BeginReceive(DataReceived, ar.AsyncState);
        }

    }
}
