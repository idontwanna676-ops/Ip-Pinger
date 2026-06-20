here is the code: using System;
using System.Net.NetworkInformation;
using System.Net.Sockets;
using System.Text;
using System.Threading;

Console.Title = "IP Pinger by wannawin";
Console.ForegroundColor = ConsoleColor.Cyan;
Console.WriteLine("========================================");
Console.WriteLine("          IP Pinger" +
    " by wannawin         ");
Console.WriteLine("========================================");
Console.ResetColor();

string host;
if (args.Length > 0)
{
    host = args[0];
}
else
{
    Console.Write("Enter IP address or Hostname to ping: ");
    host = Console.ReadLine() ?? "";
}

if (string.IsNullOrWhiteSpace(host))
{
    Console.ForegroundColor = ConsoleColor.Red;
    Console.WriteLine("Error: Hostname or IP address cannot be empty.");
    Console.ResetColor();
    return;
}

Console.Write("Enter number of pings (default 4, type 'c' for continuous): ");
string countInput = Console.ReadLine()?.Trim().ToLower() ?? "";

bool continuous = countInput == "c";
int pingCount = 4;
if (!continuous && !string.IsNullOrEmpty(countInput))
{
    if (!int.TryParse(countInput, out pingCount) || pingCount <= 0)
    {
        Console.WriteLine("Invalid input. Defaulting to 4 pings.");
        pingCount = 4;
    }
}

Console.Write("Enter timeout in ms (default 1000): ");
string timeoutInput = Console.ReadLine() ?? "";
if (!int.TryParse(timeoutInput, out int timeout) || timeout <= 0)
{
    timeout = 1000;
}

Console.WriteLine($"\nPinging {host} with 32 bytes of data:\n");

using Ping pingSender = new Ping();
PingOptions options = new PingOptions(64, true); // TTL = 64, Don't fragment

// Create a buffer of 32 bytes of data
byte[] buffer = Encoding.ASCII.GetBytes("aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa");

int successfulPings = 0;
int failedPings = 0;
long totalRoundtripTime = 0;
long minRoundtripTime = long.MaxValue;
long maxRoundtripTime = long.MinValue;

int currentPing = 0;
bool keepPinging = true;

// Handle Ctrl+C gracefully for continuous pings
Console.CancelKeyPress += (sender, e) =>
{
    e.Cancel = true; // Prevent process termination immediately
    keepPinging = false; // Stop the loop
};

while (keepPinging && (continuous || currentPing < pingCount))
{
    try
    {
        PingReply reply = pingSender.Send(host, timeout, buffer, options);

        if (reply.Status == IPStatus.Success)
        {
            Console.ForegroundColor = ConsoleColor.Green;
            Console.WriteLine($"Reply from {reply.Address}: bytes={reply.Buffer.Length} time={reply.RoundtripTime}ms TTL={reply.Options?.Ttl}");

            successfulPings++;
            totalRoundtripTime += reply.RoundtripTime;
            if (reply.RoundtripTime < minRoundtripTime) minRoundtripTime = reply.RoundtripTime;
            if (reply.RoundtripTime > maxRoundtripTime) maxRoundtripTime = reply.RoundtripTime;
        }
        else
        {
            Console.ForegroundColor = ConsoleColor.Red;
            Console.WriteLine($"Request timed out. Status: {reply.Status}");
            failedPings++;
        }
    }
    catch (PingException ex)
    {
        Console.ForegroundColor = ConsoleColor.Red;
        Console.WriteLine($"Ping failed: {ex.InnerException?.Message ?? ex.Message}");
        failedPings++;
    }
    catch (SocketException ex)
    {
        Console.ForegroundColor = ConsoleColor.Red;
        Console.WriteLine($"Socket error: {ex.Message}");
        failedPings++;
    }
    catch (Exception ex)
    {
        Console.ForegroundColor = ConsoleColor.Red;
        Console.WriteLine($"Error: {ex.Message}");
        failedPings++;
    }
    finally
    {
        Console.ResetColor();
    }

    currentPing++;
    if (keepPinging && (continuous || currentPing < pingCount))
    {
        Thread.Sleep(1000); // Wait 1 second between pings
    }
}

// Print statistics
int totalSent = successfulPings + failedPings;
if (totalSent > 0)
{
    Console.WriteLine($"\n--- Ping statistics for {host} ---");
    Console.WriteLine($"    Packets: Sent = {totalSent}, Received = {successfulPings}, Lost = {failedPings} ({((double)failedPings / totalSent) * 100:0.##}% loss)");

    if (successfulPings > 0)
    {
        double avgRoundtripTime = (double)totalRoundtripTime / successfulPings;
        Console.WriteLine("Approximate round trip times in milli-seconds:");
        Console.WriteLine($"    Minimum = {minRoundtripTime}ms, Maximum = {maxRoundtripTime}ms, Average = {avgRoundtripTime:F2}ms");
    }
}

Console.WriteLine("\nPress any key to exit...");
Console.ReadKey();
