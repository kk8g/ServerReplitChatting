import socket
import threading
import time
import os
import random
import string
from colorama import Fore, init

init(autoreset=True)

# --------------------- Config ---------------------
PASSWORD = "123456"
MAX_MSG_LENGTH = 64
PORT = 5555  # Can be changed if needed
clients = {}  # socket -> {"nick": str, "joined": str, "room": str}
rooms = {}    # room_code -> {"name": str, "members": [sockets]}
rooms["MAIN"] = {"name": "Main", "members": []}
message_count = 0

# --------------------- Utilities ---------------------
def timestamp():
    return time.strftime("%H:%M:%S", time.localtime())

def generate_room_code(length=5):
    while True:
        code = ''.join(random.choices(string.ascii_uppercase + string.digits, k=length))
        if code not in rooms:
            return code

def broadcast(message, sender_socket=None):
    for client in list(clients.keys()):
        if client != sender_socket:
            try:
                client.send(message.encode("utf-8"))
            except:
                remove_client(client)

def room_broadcast(room_code, message, exclude_socket=None):
    if room_code not in rooms:
        return
    for member in list(rooms[room_code]["members"]):
        if member != exclude_socket:
            try:
                member.send(message.encode("utf-8"))
            except:
                remove_client(member)

def remove_client(client_socket):
    if client_socket in clients:
        info = clients[client_socket]
        nick = info["nick"]
        room = info["room"]
        if room and room in rooms and client_socket in rooms[room]["members"]:
            rooms[room]["members"].remove(client_socket)
            if len(rooms[room]["members"]) == 0 and room != "MAIN":
                del rooms[room]
                print(Fore.CYAN + f"[{timestamp()}] Room '{room}' destroyed (empty).")
        del clients[client_socket]
        broadcast(f"[SYSTEM] {nick} left. Users online: {len(clients)}")
        print(Fore.RED + f"[{timestamp()}] {nick} disconnected.")
        client_socket.close()

# --------------------- Client Handler ---------------------
def handle_client(client_socket, addr):
    global message_count
    try:
        # Password check
        client_socket.send("[SYSTEM] Enter server password: ".encode("utf-8"))
        password = client_socket.recv(1024).decode("utf-8").strip()
        if password != PASSWORD:
            client_socket.send("[SYSTEM] Wrong password. Connection closed.".encode("utf-8"))
            client_socket.close()
            return
        client_socket.send("[SYSTEM] Password accepted!".encode("utf-8"))

        # Nickname
        client_socket.send("[SYSTEM] Enter your nickname: ".encode("utf-8"))
        nickname = client_socket.recv(1024).decode("utf-8").strip()
        clients[client_socket] = {"nick": nickname, "joined": timestamp(), "room": "MAIN"}
        rooms["MAIN"]["members"].append(client_socket)

        print(Fore.GREEN + f"[{timestamp()}] {nickname} joined. Users online: {len(clients)}")
        broadcast(f"[SYSTEM] {nickname} joined the Main room. Users online: {len(clients)}")

        while True:
            message = client_socket.recv(1024).decode("utf-8").strip()
            if not message:
                break

            if len(message) > MAX_MSG_LENGTH:
                client_socket.send(f"[SYSTEM] Message too long! Max {MAX_MSG_LENGTH} chars.".encode("utf-8"))
                continue

            # Commands
            if message.lower().startswith("/users"):
                user_list = [f" - {info['nick']} (joined {info['joined']}) | Room: {info['room']}" for info in clients.values()]
                client_socket.send(("[SYSTEM] Online Users:\n" + "\n".join(user_list)).encode("utf-8"))
                continue

            elif message.lower().startswith("/createroom"):
                parts = message.split(" ", 1)
                room_name = parts[1].strip() if len(parts) > 1 else "PrivateRoom"
                room_code = generate_room_code()
                rooms[room_code] = {"name": room_name, "members": [client_socket]}
                old_room = clients[client_socket]["room"]
                if old_room:
                    rooms[old_room]["members"].remove(client_socket)
                clients[client_socket]["room"] = room_code
                client_socket.send(f"[SYSTEM] Room '{room_name}' created. Join code: {room_code}".encode("utf-8"))
                continue

            elif message.lower().startswith("/joinroom"):
                parts = message.split(" ", 1)
                if len(parts) < 2:
                    client_socket.send("[SYSTEM] Usage: /joinroom <ROOMCODE>".encode("utf-8"))
                    continue
                code = parts[1].strip().upper()
                if code not in rooms:
                    client_socket.send("[SYSTEM] Room not found.".encode("utf-8"))
                    continue
                old_room = clients[client_socket]["room"]
                if old_room:
                    rooms[old_room]["members"].remove(client_socket)
                    if len(rooms[old_room]["members"]) == 0 and old_room != "MAIN":
                        del rooms[old_room]
                rooms[code]["members"].append(client_socket)
                clients[client_socket]["room"] = code
                client_socket.send(f"[SYSTEM] Joined room '{rooms[code]['name']}' ({code})".encode("utf-8"))
                room_broadcast(code, f"[SYSTEM] {nickname} joined the room.", client_socket)
                continue

            elif message.lower().startswith("/leaveroom"):
                room = clients[client_socket]["room"]
                if room != "MAIN":
                    rooms[room]["members"].remove(client_socket)
                    room_broadcast(room, f"[SYSTEM] {nickname} left the room.", client_socket)
                    client_socket.send(f"[SYSTEM] You left room '{rooms[room]['name']}'.".encode("utf-8"))
                    clients[client_socket]["room"] = "MAIN"
                    rooms["MAIN"]["members"].append(client_socket)
                    client_socket.send(f"[SYSTEM] You joined the Main room.".encode("utf-8"))
                    room_broadcast("MAIN", f"[SYSTEM] {nickname} joined the Main room.", client_socket)
                    if len(rooms[room]["members"]) == 0:
                        del rooms[room]
                else:
                    client_socket.send("[SYSTEM] You are in the Main room and cannot leave it.".encode("utf-8"))
                continue

            # Normal message
            message_count += 1
            room_code = clients[client_socket]["room"]
            log_msg = f"[{timestamp()}] {nickname}: {message}"
            print(Fore.YELLOW + log_msg)
            room_broadcast(room_code, f"[{timestamp()}] {nickname}: {message}", client_socket)

    except:
        pass
    remove_client(client_socket)

# --------------------- Server Start ---------------------
def start_server():
    os.system("cls" if os.name == "nt" else "clear")
    print(Fore.GREEN + "[STARTED] LAN Chat Server running")
    host = socket.gethostbyname(socket.gethostname())
    print(Fore.CYAN + f"[SYSTEM] Your LAN IP: {host}, port: {PORT}")
    server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    server.bind(("0.0.0.0", PORT))
    server.listen()

    while True:
        client_socket, addr = server.accept()
        threading.Thread(target=handle_client, args=(client_socket, addr)).start()

if __name__ == "__main__":
    start_server()
