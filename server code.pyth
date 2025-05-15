import socket
import threading
import json
import time
from collections import deque
from uuid import uuid4

# ====== 서버 설정 ======
SERVER_HOST = 'localhost'
SERVER_PORT = 9001
MAX_CONN = 5
BUFFER_SIZE = 4096

# ====== 데이터 저장소 ======
class MailSystem:
    def __init__(self):
        self.users = {'alice': 'pass1', 'bob': 'pass2'}
        self.inboxes = {user: deque(maxlen=100) for user in self.users}  # 최대 100개 메일 저장
        self.outbox = deque()
        self.active_connections = {}  # 실시간 연결된 사용자

    # ====== 큐 관리 ======
    def process_outbox(self):
        while self.outbox:
            mail = self.outbox.popleft()
            receiver = mail['receiver']
            
            # 실시간 전송 시도
            if receiver in self.active_connections:
                try:
                    conn = self.active_connections[receiver]
                    conn.send(json.dumps({
                        'type': 'mail_push',
                        'mail': mail
                    }).encode())
                    print(f"[큐 관리] 실시간 전송 성공: {receiver}")
                except Exception as e:
                    print(f"[큐 관리] 실시간 전송 실패: {e}")
                    self._store_to_inbox(receiver, mail)
            else:
                self._store_to_inbox(receiver, mail)

    def _store_to_inbox(self, receiver, mail):
        if receiver in self.inboxes:
            self.inboxes[receiver].append(mail)
            print(f"[큐 관리] 인박스 저장: {receiver}")
        else:
            print(f"[큐 관리] 잘못된 수신자: {receiver}")

    # ====== 사용자 요청 처리 ======
    def handle_request(self, conn, data):
        try:
            req = json.loads(data.decode())
            handler = getattr(self, f"handle_{req['type']}", None)
            return handler(conn, req) if handler else {'status': 'invalid_request'}
        except json.JSONDecodeError:
            return {'status': 'decode_error'}

    def handle_login(self, conn, req):
        if req['id'] not in self.users:
            return {'status': 'id_not_found'}
        if self.users[req['id']] != req['pw']:
            return {'status': 'wrong_password'}
        self.active_connections[req['id']] = conn
        return {'status': 'success', 'user': req['id']}

    def handle_send_mail(self, _, req):
        mail_id = str(uuid4())
        new_mail = {
            'id': mail_id,
            'sender': req['sender'],
            'receiver': req['receiver'],
            'subject': req['subject'],
            'body': req['body'],
            'timestamp': time.strftime("%Y-%m-%d %H:%M:%S")
        }
        self.outbox.append(new_mail)
        return {'status': 'queued', 'mail_id': mail_id}

    def handle_get_mail_list(self, _, req):
        user = req['user']
        return {
            'status': 'success',
            'mails': [{
                'id': mail['id'],
                'subject': mail['subject'],
                'sender': mail['sender'],
                'timestamp': mail['timestamp']
            } for mail in self.inboxes[user]]
        }

    def handle_read_mail(self, _, req):
        user = req['user']
        target_id = req['mail_id']
        for mail in self.inboxes[user]:
            if mail['id'] == target_id:
                return {'status': 'success', 'mail': mail}
        return {'status': 'mail_not_found'}

# ====== 서버 실행 ======
class MailServer:
    def __init__(self):
        self.system = MailSystem()
        self.server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        self.server_socket.bind((SERVER_HOST, SERVER_PORT))
        self.server_socket.listen(MAX_CONN)
        
        # 큐 처리 스레드 시작
        threading.Thread(target=self._process_queue, daemon=True).start()
        
        print(f"🚀 메일 서버 시작: {SERVER_HOST}:{SERVER_PORT}")

    def _process_queue(self):
        while True:
            self.system.process_outbox()
            time.sleep(1)  # 1초마다 큐 처리

    def run(self):
        while True:
            conn, addr = self.server_socket.accept()
            threading.Thread(target=self._client_handler, args=(conn,)).start()

    def _client_handler(self, conn):
        try:
            while True:
                data = conn.recv(BUFFER_SIZE)
                if not data: break
                response = self.system.handle_request(conn, data)
                conn.send(json.dumps(response).encode())
        finally:
            conn.close()

# ====== 클라이언트 ======
class MailClient:
    def __init__(self):
        self.sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        self.sock.connect((SERVER_HOST, SERVER_PORT))
        self.user = None
        
        # 실시간 알림 수신 스레드
        threading.Thread(target=self._receive_notifications, daemon=True).start()

    def _receive_notifications(self):
        while True:
            try:
                data = self.sock.recv(BUFFER_SIZE)
                if data:
                    notification = json.loads(data.decode())
                    if notification['type'] == 'mail_push':
                        print(f"\n🔔 새 메일: {notification['mail']['subject']}")
            except:
                break

    def login(self, user_id, password):
        req = {'type': 'login', 'id': user_id, 'pw': password}
        self.sock.send(json.dumps(req).encode())
        res = json.loads(self.sock.recv(BUFFER_SIZE).decode())
        if res['status'] == 'success':
            self.user = user_id
        return res

    def send_mail(self, receiver, subject, body):
        req = {
            'type': 'send_mail',
            'sender': self.user,
            'receiver': receiver,
            'subject': subject,
            'body': body
        }
        self.sock.send(json.dumps(req).encode())
        return json.loads(self.sock.recv(BUFFER_SIZE).decode())

    def get_mail_list(self):
        req = {'type': 'get_mail_list', 'user': self.user}
        self.sock.send(json.dumps(req).encode())
        return json.loads(self.sock.recv(BUFFER_SIZE).decode())

    def read_mail(self, mail_id):
        req = {'type': 'read_mail', 'user': self.user, 'mail_id': mail_id}
        self.sock.send(json.dumps(req).encode())
        return json.loads(self.sock.recv(BUFFER_SIZE).decode())

# ====== 실행 예시 ======
if __name__ == "__main__":
    import sys
    
    if '--server' in sys.argv:
        MailServer().run()
    else:
        client = MailClient()
        print("=== 메일 클라이언트 ===")
        user = input("사용자 ID: ")
        pw = input("비밀번호: ")
        
        res = client.login(user, pw)
        if res['status'] != 'success':
            print("❌ 로그인 실패")
            exit()
            
        print("명령어: send, list, read, exit")
        while True:
            cmd = input("> ").strip().lower()
            
            if cmd == 'send':
                to = input("수신자: ")
                subject = input("제목: ")
                body = input("내용: ")
                print(client.send_mail(to, subject, body))
                
            elif cmd == 'list':
                mails = client.get_mail_list()
                for idx, mail in enumerate(mails.get('mails', [])):
                    print(f"{idx+1}. {mail['subject']} ({mail['sender']})")
                    
            elif cmd == 'read':
                mail_id = input("메일 ID: ")
                print(client.read_mail(mail_id))
                
            elif cmd == 'exit':
                break
