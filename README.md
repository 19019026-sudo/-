# http_proxy_server.py
import socketserver
import socket
import threading

BUFFER_SIZE = 8192
LISTEN_PORT = 8888

class ProxyHandler(socketserver.BaseRequestHandler):
    def handle(self):
        data = self.request.recv(BUFFER_SIZE)
        if not data:
            return

        # リクエストラインを解析
        first_line = data.split(b'\n')[0]
        try:
            method, url, protocol = first_line.split()
        except ValueError:
            return

        if not url.startswith(b'http'):
            self.request.sendall(b"HTTP/1.1 400 Bad Request\r\n\r\nInvalid request")
            return

        # URLからホストとポートを抽出
        url_str = url.decode()
        host_port = url_str.split("/")[2]
        if ":" in host_port:
            host, port = host_port.split(":")
            port = int(port)
        else:
            host, port = host_port, 80

        try:
            # ターゲットサーバーへ接続
            with socket.create_connection((host, port)) as remote_socket:
                remote_socket.sendall(data)
                while True:
                    reply = remote_socket.recv(BUFFER_SIZE)
                    if not reply:
                        break
                    self.request.sendall(reply)
        except Exception as e:
            err_msg = f"HTTP/1.1 502 Bad Gateway\r\n\r\nProxy Error: {e}".encode()
            self.request.sendall(err_msg)

class ThreadedTCPServer(socketserver.ThreadingMixIn, socketserver.TCPServer):
    allow_reuse_address = True

def main():
    print(f"Starting HTTP Proxy on port {LISTEN_PORT} ...")
    with ThreadedTCPServer(("", LISTEN_PORT), ProxyHandler) as server:
        server.serve_forever()

if __name__ == "__main__":
    main()
python3 http_proxy_server.py
curl -x http://localhost:8888 http://example.com
