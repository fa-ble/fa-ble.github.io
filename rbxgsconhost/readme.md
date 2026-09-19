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

RBXGSConHostSoap.php

```php
<?php

namespace CompactInteractive;

class RBXGSConHost {
    public static $ip;
    public static $port;
    public static $url;
    public static $fullurl;
    public static array $errors = [];

    public static function haserrors() {
        if(empty(self::$errors)) {
            return true;
        }

        return false;
    }

    public static function geterrors() {
        foreach (self::$errors as $error) {
            return "Error: {$error} <br>";
        }
    }

    public static function init($ip = "127.0.0.1", $port = 64989, $url = "localhost") {
        if(!filter_var($ip, FILTER_VALIDATE_IP)) {
            self::$errors[] = "Invalid IP address given.";
        }

        if(!is_int($port) || $port < 1 || $port > 65535) {
            self::$errors[] = "Invalid port given.";
        }
        
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

        if (curl_errno($ch)) {
            $error = curl_error($ch);
            self::$errors[] = "CURL Error: {$error}";
            return false;
        }
        
        $status = curl_getinfo($ch, CURLINFO_HTTP_CODE);
    
        if ($status !== 200) {
            self::$errors[] = "CURL returned status code: {$status}";
            return false;
        }

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

If you are incapable of using this soap script, please never touch coding again
