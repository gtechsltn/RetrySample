# Retry Sample in C#

+ You should separate the IRCBot from the Main into its own class with it's own fields/properties
+ You should specify all required parameters via the construtor so that it is not possible to create an invalid ircbot
+ I don't think it's a good idea to make the retry recursive - IMHO it would be better to create a loop and limit the retries so that you don't have an infite loop (ok, unless it was desired)
+ You really should use the using statement. In your current implementation the resources will be closed only if there was no execption - if an exception occurs the Closees won't be called. You also don't free the resources because you don't call the Dispose, you just close them and then create new resources on retry, this is a memory leak.

# References

https://codereview.stackexchange.com/questions/142653/simple-irc-bot-in-c

### Main.cs
```
void Main()
{
    var ircBot = new IRCbot(
        server: "irc.changeme.com",
        port: 6667,
        user: "USER IRCbot 0 * :IRCbot",
        nick: "IRCbot",
        channel: "#opers"
    );

    ircBot.Start();
}
```

### IRCbot.cs
```
public class IRCbot
{
    // server to connect to (edit at will)
    private readonly string _server;
    // server port (6667 by default)
    private readonly int _port;
    // user information defined in RFC 2812 (IRC: Client Protocol) is sent to the IRC server 
    private readonly string _user;

    // the bot's nickname
    private readonly string _nick;
    // channel to join
    private readonly string _channel;

    private readonly int _maxRetries;

    public IRCbot(string server, int port, string user, string nick, string channel, int maxRetries = 3)
    {
        _server = server;
        _port = port;
        _user = user;
        _nick = nick;
        _channel = channel;
        _maxRetries = maxRetries;
    }

    public void Start()
    {
        var retry = false;
        var retryCount = 0;
        do
        {
            try
            {
                using (var irc = new TcpClient(_server, _port))
                using (var stream = irc.GetStream())
                using (var reader = new StreamReader(stream))
                using (var writer = new StreamWriter(stream))
                {
                    writer.WriteLine("NICK " + _nick);
                    writer.Flush();
                    writer.WriteLine(_user);
                    writer.Flush();

                    while (true)
                    {
                        string inputLine;
                        while ((inputLine = reader.ReadLine()) != null)
                        {
                            Console.WriteLine("<- " + inputLine);

                            // split the lines sent from the server by spaces (seems to be the easiest way to parse them)
                            string[] splitInput = inputLine.Split(new Char[] { ' ' });

                            if (splitInput[0] == "PING")
                            {
                                string PongReply = splitInput[1];
                                //Console.WriteLine("->PONG " + PongReply);
                                writer.WriteLine("PONG " + PongReply);
                                writer.Flush();
                                //continue;
                            }

                            switch (splitInput[1])
                            {
                                case "001":
                                    writer.WriteLine("JOIN " + _channel);
                                    writer.Flush();
                                    break;
                                default:
                                    break;
                            }
                        }
                    }
                }
            }
            catch (Exception e)
            {
                // shows the exception, sleeps for a little while and then tries to establish a new connection to the IRC server
                Console.WriteLine(e.ToString());
                Thread.Sleep(5000);
                retry = ++retryCount <= _maxRetries;
            }
        } while (retry);
    }
}
```
