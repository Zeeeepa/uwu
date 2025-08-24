# Windows Deployment Guide for uwu with DeepSeek

This guide provides step-by-step instructions for deploying the `uwu` CLI tool on Windows, configured to use DeepSeek as the AI provider.

## Prerequisites

- Windows 10 or 11
- PowerShell 5.1 or later
- Administrator privileges (for some installation steps)
- Internet connection

## 1. Install Bun Runtime

Bun is required to build and run the `uwu` tool. Install it using PowerShell:

```powershell
# Open PowerShell as Administrator and run:
powershell -Command "irm bun.sh/install.ps1 | iex"
```

After installation, close and reopen PowerShell to ensure Bun is in your PATH.

## 2. Clone the Repository

```powershell
# Clone the repository
git clone https://github.com/context-labs/uwu.git
cd uwu
```

## 3. Install Dependencies and Build

```powershell
# Install dependencies
bun install

# Build the uwu CLI tool
bun run build
```

This will create the `uwu-cli` binary in the `dist/` directory.

## 4. Make the Binary Accessible

```powershell
# Create a directory in your PATH if it doesn't exist
$binDir = "$env:USERPROFILE\bin"
if (-not (Test-Path $binDir)) {
    New-Item -ItemType Directory -Path $binDir
}

# Copy the binary to the bin directory
Copy-Item -Path ".\dist\uwu-cli" -Destination "$binDir\uwu-cli"

# Add the bin directory to your PATH if it's not already there
if ($env:PATH -notlike "*$binDir*") {
    [Environment]::SetEnvironmentVariable("PATH", "$env:PATH;$binDir", "User")
    $env:PATH = "$env:PATH;$binDir"
}
```

## 5. Configure DeepSeek Integration

Create a configuration file for `uwu` that uses DeepSeek as the AI provider:

```powershell
# Get the config directory path
$configDir = "$env:APPDATA\uwu"
if (-not (Test-Path $configDir)) {
    New-Item -ItemType Directory -Path $configDir
}

# Create the DeepSeek configuration file
$configContent = @"
{
  "type": "Custom",
  "model": "deepseek-coder",
  "baseURL": "https://api.deepseek.com/v1",
  "apiKey": "sk-915975a658dd4f61s",
  "clipboard": true,
  "context": {
    "enabled": true,
    "maxHistoryCommands": 10
  }
}
"@

# Write the configuration to the file
$configContent | Out-File -FilePath "$configDir\config.json" -Encoding utf8
```

## 6. Set Up PowerShell Integration

Add the following function to your PowerShell profile to enable seamless `uwu` integration:

```powershell
# Open your PowerShell profile for editing
if (-not (Test-Path $PROFILE)) {
    New-Item -Type File -Path $PROFILE -Force
}
notepad $PROFILE
```

Add the following function to your profile:

```powershell
function uwu {
    param(
        [Parameter(ValueFromRemainingArguments=$true)]
        $args
    )
    $Source = '
    using System;
    using System.Runtime.InteropServices;

    public class ConsoleInjector {
        [StructLayout(LayoutKind.Sequential)]
        public struct KEY_EVENT_RECORD {
            public bool bKeyDown;
            public ushort wRepeatCount;
            public ushort wVirtualKeyCode;
            public ushort wVirtualScanCode;
            public char UnicodeChar;
            public uint dwControlKeyState;
        }

        [StructLayout(LayoutKind.Sequential)]
        public struct INPUT_RECORD {
            public ushort EventType;
            public KEY_EVENT_RECORD KeyEvent;
        }

        [DllImport("kernel32.dll", SetLastError = true)]
        public static extern IntPtr GetStdHandle(int nStdHandle);

        [DllImport("kernel32.dll", SetLastError = true)]
        public static extern bool WriteConsoleInput(
            IntPtr hConsoleInput,
            INPUT_RECORD[] lpBuffer,
            int nLength,
            out int lpNumberOfEventsWritten
        );

        const int STD_INPUT_HANDLE = -10;
        const ushort KEY_EVENT = 0x0001;

        public static void SendCommand(string text) {
            IntPtr hIn = GetStdHandle(STD_INPUT_HANDLE);
            var records = new INPUT_RECORD[text.Length];
            
            int i = 0;
            for (; i < text.Length; i++) {
                records[i].EventType = KEY_EVENT;
                records[i].KeyEvent.bKeyDown = true;
                records[i].KeyEvent.wRepeatCount = 1;
                records[i].KeyEvent.UnicodeChar = text[i];
            }

            int written;
            WriteConsoleInput(hIn, records, i, out written);
        }
    }';
    $cmd = uwu-cli @args;
    Add-Type -TypeDefinition $Source;
    [ConsoleInjector]::SendCommand($cmd)
}
```

Save the file and reload your PowerShell profile:

```powershell
. $PROFILE
```

## 7. Create a Command Prompt Integration (Optional)

If you want to use `uwu` from Command Prompt (cmd.exe) as well, create a batch file:

```powershell
# Create a batch file in your bin directory
$batchContent = @"
@ECHO OFF
CHCP 437 >NUL
POWERSHELL uwu %*
@ECHO ON
"@

$batchContent | Out-File -FilePath "$binDir\uwu.bat" -Encoding ascii
```

## 8. Test the Installation

Test that `uwu` is working correctly:

```powershell
# In PowerShell
uwu list all files in the current directory
```

The tool should generate and display a command like `dir` or `ls`, which you can then execute by pressing Enter.

## Troubleshooting

### API Connection Issues

If you encounter issues connecting to the DeepSeek API, verify:

1. Your internet connection is working
2. The API key is correctly entered in the config.json file
3. The baseURL is correct for DeepSeek's API endpoint

### Command Not Found

If `uwu-cli` is not found:

1. Ensure the binary was successfully built in the `dist/` directory
2. Verify the binary was copied to a directory in your PATH
3. Restart your terminal to refresh the PATH environment variable

### PowerShell Function Not Working

If the PowerShell function doesn't work:

1. Make sure your PowerShell execution policy allows running scripts
2. Check that the function was added to your profile correctly
3. Ensure you reloaded your profile with `. $PROFILE`

## Updating

To update `uwu` to the latest version:

```powershell
cd uwu
git pull
bun install
bun run build
Copy-Item -Path ".\dist\uwu-cli" -Destination "$env:USERPROFILE\bin\uwu-cli" -Force
```

## Uninstallation

To uninstall `uwu`:

```powershell
# Remove the binary
Remove-Item "$env:USERPROFILE\bin\uwu-cli"
Remove-Item "$env:USERPROFILE\bin\uwu.bat" -ErrorAction SilentlyContinue

# Remove the configuration
Remove-Item -Recurse -Force "$env:APPDATA\uwu"

# Remove the repository if desired
# Remove-Item -Recurse -Force "path\to\uwu"
```

You'll also need to manually remove the `uwu` function from your PowerShell profile.

