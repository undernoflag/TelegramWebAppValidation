# TWA (Telegram Web App) Validator For .NET Developers

This repository (TelegramWebAppValidator class) provides a C# implementation to validate the `initData` received from Telegram Web Apps. This validator ensures that the data sent from Telegram has not been tampered with, by verifying the hash using HMAC-SHA256.

## How It Works

The algorithm follows these steps:

### 1. Parse the Query String
The `initData` received from Telegram is parsed into a query string format. This allows easy access to its individual parameters.

```csharp
var nvc = HttpUtility.ParseQueryString(inputData);
```

### 2. Extract and Remove the `hash`
The `hash` parameter is extracted from the parsed data. This is the hash sent by Telegram for verification. It is then removed from the data to prevent it from being included in the HMAC calculation.

```csharp
var clientHash = nvc["hash"];
if (string.IsNullOrEmpty(clientHash)) return false;
nvc.Remove("hash");
```

### 3. Create the Data Check String
Each key-value pair in the remaining data is concatenated into the format `key=value`. These pairs are then sorted alphabetically by their keys and joined with a newline character (`\n`).

```csharp
var dataCheckList = new List<string>();
foreach (var key in nvc.AllKeys)
{
    if (!string.IsNullOrEmpty(key))
    {
        dataCheckList.Add($"{key}={nvc[key]}");
    }
}
dataCheckList.Sort(StringComparer.Ordinal);
var dataCheckString = string.Join("\n", dataCheckList);
```

### 4. Compute the Secret
The `botToken` provided to your bot by Telegram is used to compute the secret key. This is achieved using HMAC-SHA256 with a fixed string `WebAppData` as the key.

```csharp
var secret = ComputeHmacSha256("WebAppData", botToken);
```

### 5. Compute the Final Hash
The `dataCheckString` is hashed using HMAC-SHA256, with the previously computed secret as the key.

```csharp
var finalHash = ComputeHmacSha256(secret, dataCheckString);
```

### 6. Compare the Final Hash
The computed hash is converted to a hexadecimal string and compared with the `hash` received from Telegram. Both hashes are converted to lowercase before comparison.

```csharp
var finalHashHex = ByteArrayToHexString(finalHash).ToLowerInvariant();
return finalHashHex == clientHash.ToLowerInvariant();
```

---

## Full Implementation

Here is the complete implementation of the Telegram Web App Validator:

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Web;
using System.Security.Cryptography;
using System.Text;

public static class TelegramWebAppValidator
{
    public static bool ValidateInitData(string inputData, string botToken)
    {
        var nvc = HttpUtility.ParseQueryString(inputData);
        var clientHash = nvc["hash"];
        if (string.IsNullOrEmpty(clientHash)) return false;
        nvc.Remove("hash");

        var dataCheckList = new List<string>();
        foreach (var key in nvc.AllKeys)
        {
            if (!string.IsNullOrEmpty(key))
            {
                dataCheckList.Add($"{key}={nvc[key]}");
            }
        }
        dataCheckList.Sort(StringComparer.Ordinal);
        var dataCheckString = string.Join("\n", dataCheckList);

        var secret = ComputeHmacSha256("WebAppData", botToken);
        var finalHash = ComputeHmacSha256(secret, dataCheckString);

        var finalHashHex = ByteArrayToHexString(finalHash).ToLowerInvariant();
        return finalHashHex == clientHash.ToLowerInvariant();
    }

    private static byte[] ComputeHmacSha256(string keyString, string message)
    {
        var keyBytes = Encoding.UTF8.GetBytes(keyString);
        var messageBytes = Encoding.UTF8.GetBytes(message);
        using var hmac = new HMACSHA256(keyBytes);
        return hmac.ComputeHash(messageBytes);
    }

    private static byte[] ComputeHmacSha256(byte[] keyBytes, string message)
    {
        var messageBytes = Encoding.UTF8.GetBytes(message);
        using var hmac = new HMACSHA256(keyBytes);
        return hmac.ComputeHash(messageBytes);
    }

    private static string ByteArrayToHexString(byte[] bytes)
    {
        var sb = new StringBuilder(bytes.Length * 2);
        foreach (var b in bytes)
            sb.Append(b.ToString("x2"));
        return sb.ToString();
    }
}
```

---

## How to Use

1. Include this class in your .NET project.
2. Call the `ValidateInitData` method with the `initData` and your bot token:

```csharp
bool isValid = TelegramWebAppValidator.ValidateInitData(initData, botToken);
if (isValid)
{
    Console.WriteLine("Data is valid.");
}
else
{
    Console.WriteLine("Data validation failed.");
}
```

## Related Resources

For more information on initializing Telegram Web Apps, check the [official Telegram documentation](https://core.telegram.org/bots/webapps#initializing-mini-apps).

