Sep 15 2026 1:21 PM

RBXGSConHost is a runtime environment for RBXGS, allowing it to be ran standalone without IIS (effectively making it work like RCCService).

Made by pizzaboxer and it is open-sourced


Source:
https://github.com/pizzaboxer/rbxgsconhost


Backup source:
https://github.com/fa-ble/fa-ble.github.io/tree/main/rbxgsconhost/source


Pre-compiled version:
https://github.com/hamounvg/RBXGS


Usage:
```
  -h, --help            Print this help message
  -p, --port <port>     Specify web server port (default is 64989)
  -b, --baseDir <path>  Specify path where RBXGS is located (default is working dir)
```


RBXGSConHost helps 2008-2009 revivals to effectively render, host servers without patching or client/studio hosting.

# How to use RBXGSConHost with SoapUI:

Open up SoapUI

Check RBXGS mode

Put in your IP and Port (eg: 127.0.0.1:64989)

Open a Environment (kind of like a JobID)

it will spit back a string

Pick Execute, put in the Environment ID and put in your render lua script of choice 

It will spit out Base64, decrypt it like normal

# How to (skid) this on your 2008-2009vival:

I've made a RBXGSConHost Soap Handling php script,

do not use this under any circumstances to sell/profit or gatekeep, this should be public knowledge for those who seeks to know

RBXGSConHostSoap.php

```php
<?php

namespace CompactInteractive;

class RBXGSConHost {
    public static $ip;
    public static $port;
    public static $url;
    public static $fullurl;

    // Error handling
    public static array $errors = [];

    public static function hasErrors() {
        if(empty(self::$errors)) {
            return true;
        }

        return false;
    }

    public static function getErrors() {
        foreach (self::$errors as $error) {
            return "Error: {$error} <br>";
        }
    }

    public static function init($ip = "127.0.0.1", $port = 64989, $url = "localhost") {
        // ip check
        if(!filter_var($ip, FILTER_VALIDATE_IP)) {
            self::$errors[] = "Invalid IP address given.";
        }

        // port check
        if(!is_int($port) || $port < 1 || $port > 65535) {
            self::$errors[] = "Invalid port given.";
        }
        
        // url check
        if(str_starts_with($url, 'http://') || str_starts_with($url, 'https://')) {
            // remove the http or https
            $finalurl = str_replace(['http://', 'https://'], '', $url);
            $finalurl = rtrim($finalurl, '/');
        }else {
            $finalurl = $url;
        }

        if(empty(self::$errors)) {
            self::$ip = $ip;
            self::$port = $port;
            self::$url = $finalurl;
            self::$fullurl = "http://{$ip}:{$port}";

            return true;
        }

        return false;
    }

    public static function request($xml) {
        $ch = curl_init(self::$fullurl);

        curl_setopt($ch, CURLOPT_HTTPHEADER, [ "Content-Type: text/xml" ]);
        curl_setopt($ch, CURLOPT_POST, true);
        curl_setopt($ch, CURLOPT_POSTFIELDS, $xml);
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
        curl_setopt($ch, CURLOPT_SSL_VERIFYHOST, false);
        curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, false);

        $response = curl_exec($ch);

        // Curl error handling
        if (curl_errno($ch)) {
            $error = curl_error($ch);
            self::$errors[] = "CURL Error: {$error}";
            return false;
        }
        // Status code check
        $status = curl_getinfo($ch, CURLINFO_HTTP_CODE);
    
        if ($status !== 200) {
            self::$errors[] = "CURL returned status code: {$status}";
            return false;
        }

        // Remove the gunk from RCC
        $result = str_replace(
            [
                "<ns1:value>",
                "</ns1:value>",
                "</ns1:OpenJobResult>",
                "<ns1:OpenJobResult>",
                "<ns1:type>",
                "</ns1:type>",
                "<ns1:table>",
                "</ns1:table>",
                "</ns1:OpenJobResult>",
                "</ns1:OpenJobResponse>",
                "</SOAP-ENV:Body>",
                "</SOAP-ENV:Envelope>"
            ],
            "",
            strstr(
                str_replace(
                    [
                        "LUA_TSTRING",
                        "LUA_TNUMBER",
                        "LUA_TBOOLEAN",
                        "LUA_TTABLE"
                    ],
                    "",
                    $response
                ),
                "<ns1:value>"
            )
        );

        $valuepos = strpos($result, "<ns1:LuaValue>");

        if ($valuepos !== false) {
            $result = substr($result, 0, $valuepos);
        }

        // dunno what this does but it looks cool
        clearstatcache();

        return $result;
    }

    public static function OpenEnvironment() {
        $xml = '<?xml version="1.0" encoding="UTF-8"?><soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:xsd="http://www.w3.org/2001/XMLSchema" xmlns:soapenc="http://schemas.xmlsoap.org/soap/encoding/" xmlns:tns="urn:Roblox"><soap:Body><tns:OpenEnvironment/></soap:Body></soap:Envelope>';

        $ch = curl_init(self::$fullurl);
        curl_setopt($ch, CURLOPT_HTTPHEADER, array("Content-Type: text/xml; charset=utf-8", "SOAPAction: OpenEnvironment"));
        curl_setopt($ch, CURLOPT_POST, true);
        curl_setopt($ch, CURLOPT_POSTFIELDS, $xml);
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
        curl_setopt($ch, CURLOPT_TIMEOUT, 3);

        $response = curl_exec($ch);

        if (curl_errno($ch)) {
            $error = curl_error($ch);
            self::$errors[] = "CURL Error: {$error}";
            curl_close($ch);
            return false;
        }

        curl_close($ch);

        preg_match('/<return>(.*?)<\/return>/', $response, $matches);
        return $matches[1] ?? null;
    }

    public static function Execute($envID, $script) {
        $script = htmlspecialchars($script);
        $xml = '<?xml version="1.0" encoding="UTF-8"?><soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:xsd="http://www.w3.org/2001/XMLSchema" xmlns:soapenc="http://schemas.xmlsoap.org/soap/encoding/" xmlns:tns="urn:Roblox"><soap:Body><tns:Execute><tns:environmentID>' . $envID . '</tns:environmentID><tns:script xsi:type="xsd:string">' . $script . '</tns:script></tns:Execute></soap:Body></soap:Envelope>';
        $xml = str_replace(["\r", "\n", "  "], "", $xml);

        $ch = curl_init(self::$fullurl);
        curl_setopt($ch, CURLOPT_HTTPHEADER, array("Content-Type: text/xml; charset=utf-8", "SOAPAction: Execute"));
        curl_setopt($ch, CURLOPT_POST, true);
        curl_setopt($ch, CURLOPT_POSTFIELDS, $xml);
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);

        $response = curl_exec($ch);

        if (curl_errno($ch)) {
            $error = curl_error($ch);
            self::$errors[] = "CURL Error: {$error}";
            curl_close($ch);
            return false;
        }

        curl_close($ch);

        // Extract base64 image data from SOAP response
        preg_match('/<value>(.*?)<\/value>/', $response, $matches);
        return $matches[1] ?? $response;
    }

    public static function CloseEnvironment($envID) {
        $xml = '<?xml version="1.0" encoding="UTF-8"?><soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:xsd="http://www.w3.org/2001/XMLSchema" xmlns:soapenc="http://schemas.xmlsoap.org/soap/encoding/" xmlns:tns="urn:Roblox"><soap:Body><tns:CloseEnvironment><tns:environmentID>' . $envID . '</tns:environmentID></tns:CloseEnvironment></soap:Body></soap:Envelope>';

        $ch = curl_init(self::$fullurl);
        curl_setopt($ch, CURLOPT_HTTPHEADER, array("Content-Type: text/xml; charset=utf-8", "SOAPAction: CloseEnvironment"));
        curl_setopt($ch, CURLOPT_POST, true);
        curl_setopt($ch, CURLOPT_POSTFIELDS, $xml);
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);

        $response = curl_exec($ch);

        if (curl_errno($ch)) {
            $error = curl_error($ch);
            self::$errors[] = "CURL Error: {$error}";
            curl_close($ch);
            return false;
        }

        curl_close($ch);

        return $response;
    }

    public static function HelloWorld() {
        $xml = '<?xml version="1.0" encoding="UTF-8"?><soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:xsd="http://www.w3.org/2001/XMLSchema" xmlns:soapenc="http://schemas.xmlsoap.org/soap/encoding/" xmlns:tns="urn:Roblox"><soap:Body><tns:HelloWorld/></soap:Body></soap:Envelope>';

        $ch = curl_init(self::$fullurl);
        curl_setopt($ch, CURLOPT_HTTPHEADER, array("Content-Type: text/xml; charset=utf-8", "SOAPAction: HelloWorld"));
        curl_setopt($ch, CURLOPT_POST, true);
        curl_setopt($ch, CURLOPT_POSTFIELDS, $xml);
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);

        $response = curl_exec($ch);

        if (curl_errno($ch)) {
            $error = curl_error($ch);
            self::$errors[] = "CURL Error: {$error}";
            curl_close($ch);
            return false;
        }

        curl_close($ch);

        return $response;
    }

    public static function GetAllEnvironments() {
        $xml = '<?xml version="1.0" encoding="UTF-8"?><soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:xsd="http://www.w3.org/2001/XMLSchema" xmlns:soapenc="http://schemas.xmlsoap.org/soap/encoding/" xmlns:tns="urn:Roblox"><soap:Body><tns:GetAllEnvironments/></soap:Body></soap:Envelope>';

        $ch = curl_init(self::$fullurl);
        curl_setopt($ch, CURLOPT_HTTPHEADER, array("Content-Type: text/xml; charset=utf-8", "SOAPAction: GetAllEnvironments"));
        curl_setopt($ch, CURLOPT_POST, true);
        curl_setopt($ch, CURLOPT_POSTFIELDS, $xml);
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);

        $response = curl_exec($ch);

        if (curl_errno($ch)) {
            $error = curl_error($ch);
            self::$errors[] = "CURL Error: {$error}";
            curl_close($ch);
            return false;
        }

        curl_close($ch);

        return $response;
    }

    public static function CloseAllEnvironments() {
        $xml = '<?xml version="1.0" encoding="UTF-8"?><soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:xsd="http://www.w3.org/2001/XMLSchema" xmlns:soapenc="http://schemas.xmlsoap.org/soap/encoding/" xmlns:tns="urn:Roblox"><soap:Body><tns:CloseAllEnvironments/></soap:Body></soap:Envelope>';

        $ch = curl_init(self::$fullurl);
        curl_setopt($ch, CURLOPT_HTTPHEADER, array("Content-Type: text/xml; charset=utf-8", "SOAPAction: CloseAllEnvironments"));
        curl_setopt($ch, CURLOPT_POST, true);
        curl_setopt($ch, CURLOPT_POSTFIELDS, $xml);
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);

        $response = curl_exec($ch);

        if (curl_errno($ch)) {
            $error = curl_error($ch);
            self::$errors[] = "CURL Error: {$error}";
            curl_close($ch);
            return false;
        }

        curl_close($ch);

        return $response;
    }

    public static function GetStatus() {
        $xml = '<?xml version="1.0" encoding="UTF-8"?><soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:xsd="http://www.w3.org/2001/XMLSchema" xmlns:soapenc="http://schemas.xmlsoap.org/soap/encoding/" xmlns:tns="urn:Roblox"><soap:Body><tns:GetStatus/></soap:Body></soap:Envelope>';

        $ch = curl_init(self::$fullurl);
        curl_setopt($ch, CURLOPT_HTTPHEADER, array("Content-Type: text/xml; charset=utf-8", "SOAPAction: GetStatus"));
        curl_setopt($ch, CURLOPT_POST, true);
        curl_setopt($ch, CURLOPT_POSTFIELDS, $xml);
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);

        $response = curl_exec($ch);

        if (curl_errno($ch)) {
            $error = curl_error($ch);
            self::$errors[] = "CURL Error: {$error}";
            curl_close($ch);
            return false;
        }

        curl_close($ch);

        return $response;
    }

    public static function GetVersion() {
        $xml = '<?xml version="1.0" encoding="UTF-8"?><soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:xsd="http://www.w3.org/2001/XMLSchema" xmlns:soapenc="http://schemas.xmlsoap.org/soap/encoding/" xmlns:tns="urn:Roblox"><soap:Body><tns:GetVersion/></soap:Body></soap:Envelope>';

        $ch = curl_init(self::$fullurl);
        curl_setopt($ch, CURLOPT_HTTPHEADER, array("Content-Type: text/xml; charset=utf-8", "SOAPAction: GetVersion"));
        curl_setopt($ch, CURLOPT_POST, true);
        curl_setopt($ch, CURLOPT_POSTFIELDS, $xml);
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);

        $response = curl_exec($ch);

        if (curl_errno($ch)) {
            $error = curl_error($ch);
            self::$errors[] = "CURL Error: {$error}";
            curl_close($ch);
            return false;
        }

        curl_close($ch);

        preg_match('/<return>(.*?)<\/return>/', $response, $matches);
        return $matches[1] ?? null;
    }
}
```


Render.php

```php
<?php
require_once($_SERVER["DOCUMENT_ROOT"] . "/inc/config.php");
require_once($_SERVER["DOCUMENT_ROOT"] . "/Assemblies/Roblox/Grid/Rcc/RBXGSConHost.php");

use CompactInteractive\RBXGSConHost;

if ($isloggedin !== 'yes') {
    header('location: /');
    exit;
}

$id = $_GET['ID'] ?? $_USER['id'];

try {
    $sql = $conn->prepare("SELECT * FROM users WHERE id = :id");
    $sql->bindParam(':id', $id, PDO::PARAM_INT);
    $sql->execute();
    $user = $sql->fetch();

    if (!$user) {
        header("Location: /My/Character.aspx");
        exit;
    }

    $avatarFilePath = $_SERVER['DOCUMENT_ROOT'] . "/Thumbs/" . $id . ".png";

    // Initialize RBXGS
    RBXGSConHost::init("127.0.0.1", 64989, "localhost");

    $headColor = (int)($user['headcolor'] ?? 24); // Yellow
    $leftArmColor = (int)($user['leftarmcolor'] ?? 24); // Yellow
    $rightArmColor = (int)($user['rightarmcolor'] ?? 24); // Yellow
    $leftLegColor = (int)($user['leftlegcolor'] ?? 23); // Blue
    $rightLegColor = (int)($user['rightlegcolor'] ?? 23); // Blue
    $torsoColor = (int)($user['torsocolor'] ?? 2); // Red

    $tshirtId = (int)($user['tshirt'] ?? 0);
    $tshirtAsset = '';
    if ($tshirtId > 0) {
        $stmt = $conn->prepare("SELECT asset FROM catalog WHERE id = ? AND type = 'tshirt'");
        $stmt->execute([$tshirtId]);
        $tshirtData = $stmt->fetch();
        if ($tshirtData) {
            $tshirtAsset = $tshirtData['asset'];
        }
    }

    $luaCode = 'local player = game:GetService("Players"):CreateLocalPlayer(0) ';
    $luaCode .= 'player:LoadCharacter(0) ';
    $luaCode .= 'local char = player.Character or player.CharacterAdded:Wait() ';
    $luaCode .= 'local bodyColors = Instance.new("BodyColors", char) ';
    $luaCode .= 'bodyColors.HeadColor = BrickColor.new(' . $headColor . ') ';
    $luaCode .= 'bodyColors.TorsoColor = BrickColor.new(' . $torsoColor . ') ';
    $luaCode .= 'bodyColors.LeftArmColor = BrickColor.new(' . $leftArmColor . ') ';
    $luaCode .= 'bodyColors.RightArmColor = BrickColor.new(' . $rightArmColor . ') ';
    $luaCode .= 'bodyColors.LeftLegColor = BrickColor.new(' . $leftLegColor . ') ';
    $luaCode .= 'bodyColors.RightLegColor = BrickColor.new(' . $rightLegColor . ') ';
    
    if ($tshirtAsset) {
        $luaCode .= 'local shirt = Instance.new("Shirt", char) ';
        $luaCode .= 'shirt.ShirtTemplate = "' . $tshirtAsset . '" ';
    }
    
    $luaCode .= 'local camera = Instance.new("Camera") ';
    $luaCode .= 'camera.CoordinateFrame = CFrame.new(0, 1.5, 8, 0, 0, -1, 0, 1, 0, 1, 0, 0) ';
    $luaCode .= 'workspace.CurrentCamera = camera ';
    $luaCode .= 'local thumbGen = game:GetService("ThumbnailGenerator") ';
    $luaCode .= 'local result = thumbGen:Click("PNG", 420, 420, true) ';
    $luaCode .= 'return result';

    // Open environment
    $envID = RBXGSConHost::OpenEnvironment();
    error_log("Render attempt for user $id: OpenEnvironment returned: " . ($envID ?: 'false/empty'));

    if ($envID) {
        // Execute script with environment ID
        $response = RBXGSConHost::Execute($envID, $luaCode);
        error_log("Render attempt for user $id: Execute returned response length: " . strlen($response));

        // Extract base64 image from response
        preg_match('/(iVBOR[\w\+\/=]+)/', $response, $imgMatches);
        if (isset($imgMatches[1])) {
            $decoded = base64_decode($imgMatches[1]);
            file_put_contents($avatarFilePath, $decoded);
            error_log("RBXGS render successful for user $id, saved to $avatarFilePath");
        } else {
            error_log("RBXGS render failed - no image data in response for user $id. Response: " . substr($response, 0, 1000));
        }

        // Close environment
        RBXGSConHost::CloseEnvironment($envID);
        error_log("Render attempt for user $id: Closed environment $envID");
    } else {
        error_log("RBXGS render failed - could not open environment for user $id");
    }

    header("Location: /My/Character.aspx");
    exit;
} catch (Exception $e) {
    error_log("General Error: " . $e->getMessage());
    exit;
}
```

To use the soap correctly, I've prepared you a graphical example PHP script

Debug_Render.php

```php
<?php
$isDebugMode = isset($_COOKIE['debug']) && $_COOKIE['debug'] === 'true';
if (!$isDebugMode) {
    die("This page is only accessible in debug mode.");
}

require_once($_SERVER["DOCUMENT_ROOT"]."/Assemblies/Roblox/Grid/Rcc/RBXGSConHost.php");

use CompactInteractive\RBXGSConHost;

$debug_info = [];
$errors = [];
$success = false;

try {
    RBXGSConHost::init("127.0.0.1", 64989, "localhost");
    $debug_info['rcc_config'] = ['ip' => '127.0.0.1', 'port' => 64989, 'url' => 'localhost'];
    $versionResult = RBXGSConHost::GetVersion();
    $debug_info['version'] = $versionResult;
    $testEnv = RBXGSConHost::OpenEnvironment();
    $debug_info['env_id'] = $testEnv;
    if (!empty($testEnv)) {
        $debug_info['rcc_status'] = 'CONNECTED';
        RBXGSConHost::CloseEnvironment($testEnv);
    } else {
        $debug_info['rcc_status'] = 'CONNECTED_BUT_EMPTY_RESPONSE';
        $errors[] = "RCC connected but returned empty response";
    }
} catch (Exception $e) {
    $debug_info['rcc_status'] = 'FAILED';
    $debug_info['rcc_error'] = $e->getMessage();
    $errors[] = "RCC Connection failed: " . $e->getMessage();
}

if ($_SERVER["REQUEST_METHOD"] == "POST" && isset($_POST['render'])) {
    try {
        $headColor    = $_POST['headcolor'];
        $torsoColor   = $_POST['torsocolor'];
        $leftArmColor  = $_POST['leftarmcolor'];
        $rightArmColor = $_POST['rightarmcolor'];
        $leftLegColor  = $_POST['leftlegcolor'];
        $rightLegColor = $_POST['rightlegcolor'];

        $debug_info['render_colors'] = [
            'head'     => $headColor,
            'torso'    => $torsoColor,
            'leftarm'  => $leftArmColor,
            'rightarm' => $rightArmColor,
            'leftleg'  => $leftLegColor,
            'rightleg' => $rightLegColor
        ];

        $thumbnailScript = <<<LUA
local player = game:GetService("Players"):CreateLocalPlayer(0)
player:LoadCharacter(0)

local char = player.Character or player.CharacterAdded:Wait()

local bodyColors = Instance.new("BodyColors", char)
bodyColors.HeadColor = BrickColor.new($headColor)
bodyColors.LeftArmColor = BrickColor.new($leftArmColor)
bodyColors.RightArmColor = BrickColor.new($rightArmColor)
bodyColors.LeftLegColor = BrickColor.new($leftLegColor)
bodyColors.RightLegColor = BrickColor.new($rightLegColor)
bodyColors.TorsoColor = BrickColor.new($torsoColor)

local camera = Instance.new("Camera")
camera.CoordinateFrame = CFrame.new(0, 1.5, 8, 0, 0, -1, 0, 1, 0, 1, 0, 0)
workspace.CurrentCamera = camera

local thumbGen = game:GetService("ThumbnailGenerator")
local result = thumbGen:Click("PNG", 400, 400, true)
return result
LUA;

        $debug_info['lua_script'] = $thumbnailScript;
        $debug_info['lua_script_length'] = strlen($thumbnailScript);

        $renderEnvID = RBXGSConHost::OpenEnvironment();
        if($renderEnvID) {
            $render = RBXGSConHost::Execute($renderEnvID, $thumbnailScript);
            RBXGSConHost::CloseEnvironment($renderEnvID);
        }

        $debug_info['render_result_raw'] = $render;
        $debug_info['render_result_length'] = strlen($render);

        if (empty($render)) {
            $errors[] = "RCC returned empty render result";
            $debug_info['render_status'] = 'FAILED_EMPTY';
        } else {
            $debug_info['render_status'] = 'SUCCESS';
            $success = true;

            preg_match('/(iVBOR[\w\+\/=]+)/', $render, $imgMatches);
            if (isset($imgMatches[1])) {
                $debug_info['thumbnail_base64'] = $imgMatches[1];
                $debug_info['base64_length'] = strlen($imgMatches[1]);
            } else {
                $debug_info['extraction_error'] = 'No base64 match found';
                $debug_info['response_preview'] = substr($render, 0, 500);
            }

            // Database update removed - no database handling
        }
    } catch (Exception $e) {
        $errors[] = "Render error: " . $e->getMessage();
        $debug_info['render_error'] = $e->getMessage();
    }
}

// Default colors instead of database query
if (!isset($debug_info['user_colors'])) {
    $debug_info['user_colors'] = [
        'headcolor'     => $_POST['headcolor'] ?? '24',
        'torsocolor'    => $_POST['torsocolor'] ?? '2',
        'leftarmcolor'  => $_POST['leftarmcolor'] ?? '24',
        'rightarmcolor' => $_POST['rightarmcolor'] ?? '24',
        'leftlegcolor'  => $_POST['leftlegcolor'] ?? '23',
        'rightlegcolor' => $_POST['rightlegcolor'] ?? '23'
    ];
}
?>
<!DOCTYPE html>
<html>
<head>
    <title>Avatar Rendering Debug (No Database)</title>
    <style>
        body { font-family: monospace; padding: 20px; background: #f0f0f0; }
        .section { background: white; padding: 15px; margin: 10px 0; border: 1px solid #ccc; }
        .success { color: green; font-weight: bold; }
        .error { color: red; font-weight: bold; }
        .info { color: blue; }
        pre { background: #f5f5f5; padding: 10px; overflow-x: auto; }
        table { border-collapse: collapse; width: 100%; }
        th, td { border: 1px solid #ddd; padding: 8px; text-align: left; }
        th { background: #f0f0f0; }
        input[type="number"] { width: 60px; }
        button { padding: 10px 20px; background: #4CAF50; color: white; border: none; cursor: pointer; }
        button:hover { background: #45a049; }
    </style>
</head>
<body>
    <div class="section">
        <h2>RBXGSConHost Renderer (No Database)</h2>
        <p>Status: <span class="<?php echo $debug_info['rcc_status'] === 'CONNECTED' ? 'success' : 'error'; ?>">
            <?php echo $debug_info['rcc_status']; ?>
        </span></p>
        <?php if (isset($debug_info['rcc_error'])): ?>
            <p class="error">Error: <?php echo htmlspecialchars($debug_info['rcc_error']); ?></p>
        <?php endif; ?>
        <p>Version: <?php echo htmlspecialchars($debug_info['version']); ?></p>
        <p>Environment: <?php echo htmlspecialchars($debug_info['env_id']); ?></p>
    </div>

    <div class="section">
        <h2>Current Body Colors</h2>
        <table>
            <tr><th>Body Part</th><th>Color ID</th></tr>
            <tr><td>Head</td><td><?php echo $debug_info['user_colors']['headcolor']; ?></td></tr>
            <tr><td>Torso</td><td><?php echo $debug_info['user_colors']['torsocolor']; ?></td></tr>
            <tr><td>Left Arm</td><td><?php echo $debug_info['user_colors']['leftarmcolor']; ?></td></tr>
            <tr><td>Right Arm</td><td><?php echo $debug_info['user_colors']['rightarmcolor']; ?></td></tr>
            <tr><td>Left Leg</td><td><?php echo $debug_info['user_colors']['leftlegcolor']; ?></td></tr>
            <tr><td>Right Leg</td><td><?php echo $debug_info['user_colors']['rightlegcolor']; ?></td></tr>
        </table>
    </div>

    <div class="section">
        <h2>Test Avatar Render</h2>
        <form method="POST">
            <table>
                <tr><td>Head Color:</td><td><input type="number" name="headcolor" id="headcolor" value="<?php echo $debug_info['user_colors']['headcolor']; ?>"> <input type="color" id="headcolor_picker" value="#ffffff" onchange="updateColor('headcolor')"></td></tr>
                <tr><td>Torso Color:</td><td><input type="number" name="torsocolor" id="torsocolor" value="<?php echo $debug_info['user_colors']['torsocolor']; ?>"> <input type="color" id="torsocolor_picker" value="#ffffff" onchange="updateColor('torsocolor')"></td></tr>
                <tr><td>Left Arm Color:</td><td><input type="number" name="leftarmcolor" id="leftarmcolor" value="<?php echo $debug_info['user_colors']['leftarmcolor']; ?>"> <input type="color" id="leftarmcolor_picker" value="#ffffff" onchange="updateColor('leftarmcolor')"></td></tr>
                <tr><td>Right Arm Color:</td><td><input type="number" name="rightarmcolor" id="rightarmcolor" value="<?php echo $debug_info['user_colors']['rightarmcolor']; ?>"> <input type="color" id="rightarmcolor_picker" value="#ffffff" onchange="updateColor('rightarmcolor')"></td></tr>
                <tr><td>Left Leg Color:</td><td><input type="number" name="leftlegcolor" id="leftlegcolor" value="<?php echo $debug_info['user_colors']['leftlegcolor']; ?>"> <input type="color" id="leftlegcolor_picker" value="#ffffff" onchange="updateColor('leftlegcolor')"></td></tr>
                <tr><td>Right Leg Color:</td><td><input type="number" name="rightlegcolor" id="rightlegcolor" value="<?php echo $debug_info['user_colors']['rightlegcolor']; ?>"> <input type="color" id="rightlegcolor_picker" value="#ffffff" onchange="updateColor('rightlegcolor')"></td></tr>
            </table>
            <br>
            <button type="submit" name="render" value="1">Render Avatar</button>
        </form>
    </div>

    <?php if ($_SERVER["REQUEST_METHOD"] == "POST" && isset($_POST['render'])): ?>
    <div class="section">
        <h2>Render Results</h2>
        <p>Status: <span class="<?php echo $success ? 'success' : 'error'; ?>">
            <?php echo $debug_info['render_status']; ?>
        </span></p>

        <?php if ($success): ?>
        <h3>Generated Thumbnail</h3>
        <?php if (isset($debug_info['thumbnail_base64'])): ?>
        <img src="data:image/png;base64,<?php echo $debug_info['thumbnail_base64']; ?>" style="border: 1px solid #000; max-width: 400px;">
        <p>Base64 length: <?php echo $debug_info['base64_length']; ?> bytes</p>
        <?php else: ?>
        <p class="error">Failed to extract base64 image</p>
        <?php if (isset($debug_info['extraction_error'])): ?>
        <p>Error: <?php echo $debug_info['extraction_error']; ?></p>
        <?php endif; ?>
        <?php if (isset($debug_info['response_preview'])): ?>
        <p>Response preview: <?php echo htmlspecialchars($debug_info['response_preview']); ?></p>
        <?php endif; ?>
        <?php endif; ?>
        <?php endif; ?>
    </div>
    <?php endif; ?>

    <?php if (!empty($errors)): ?>
    <div class="section">
        <h2 class="error">Errors</h2>
        <ul>
            <?php foreach ($errors as $error): ?>
                <li class="error"><?php echo htmlspecialchars($error); ?></li>
            <?php endforeach; ?>
        </ul>
    </div>
    <?php endif; ?>

<script>
const brickColorMap = {
    '#F2F3F3': 1, '#E5E4DF': 208, '#A3A2A5': 194, '#635F62': 199,
    '#1B2A35': 26, '#C4281C': 21, '#F5CD30': 24, '#FDEA8D': 226,
    '#0D69AC': 23, '#008F9C': 107, '#6E99CA': 102, '#80BBDB': 11,
    '#B4D2E4': 45, '#74869D': 135, '#DA8541': 106, '#E29B40': 105,
    '#27462D': 141, '#287F47': 28, '#4B974B': 37, '#A4BD47': 119,
    '#A1C48C': 29, '#789082': 210, '#A05F35': 38, '#694028': 192,
    '#6B327C': 104, '#E8BAC8': 9, '#DA867A': 101, '#D7C59A': 5,
    '#957977': 153, '#7C5C46': 217, '#CC8E69': 18, '#EAB892': 125
};

const reverseBrickColorMap = Object.fromEntries(
    Object.entries(brickColorMap).map(([hex, id]) => [id, hex])
);

function updateColor(field) {
    const picker = document.getElementById(field + '_picker');
    const input = document.getElementById(field);
    const hex = picker.value.toUpperCase();

    if (brickColorMap[hex]) {
        input.value = brickColorMap[hex];
    } else {
        let closestColor = null;
        let minDistance = Infinity;

        for (const [mapHex, id] of Object.entries(brickColorMap)) {
            const distance = colorDistance(hex, mapHex);
            if (distance < minDistance) {
                minDistance = distance;
                closestColor = id;
            }
        }

        if (closestColor) {
            input.value = closestColor;
        }
    }
}

function colorDistance(hex1, hex2) {
    const r1 = parseInt(hex1.substr(1, 2), 16);
    const g1 = parseInt(hex1.substr(3, 2), 16);
    const b1 = parseInt(hex1.substr(5, 2), 16);

    const r2 = parseInt(hex2.substr(1, 2), 16);
    const g2 = parseInt(hex2.substr(3, 2), 16);
    const b2 = parseInt(hex2.substr(5, 2), 16);

    return Math.sqrt(
        Math.pow(r1 - r2, 2) +
        Math.pow(g1 - g2, 2) +
        Math.pow(b1 - b2, 2)
    );
}

function initPickers() {
    const fields = ['headcolor', 'torsocolor', 'leftarmcolor', 'rightarmcolor', 'leftlegcolor', 'rightlegcolor'];
    fields.forEach(field => {
        const input = document.getElementById(field);
        const picker = document.getElementById(field + '_picker');
        const id = parseInt(input.value);
        if (reverseBrickColorMap[id]) {
            picker.value = reverseBrickColorMap[id];
        }
    });
}

initPickers();
</script>

</body>
</html>
```
