
This question was a bit difficult for me to understand at first 

Problem Description :
GLaDOS has locked the flag behind twenty Aperture Science test chambers. Each chamber is an illuminated panel grid — pressing any panel toggles it and all its immediate neighbours. Find the sequence of presses that extinguishes every panel in each chamber, and she will release what you've earned.

This was our target host 
http://67.205.179.14:9000/

Lets visit it 

![[Pasted image 20260619235505.png]]

#### How it works

- You have a grid of panels (0=off, 1=on)
- Pressing a cell toggles it **and its 4 orthogonal neighbors**
- Goal: make all cells **0**
- You need to do this for **20 chambers** and send solutions over a socket

#### Solving Strategy: Gaussian Elimination over GF(2)

Each cell press is a binary variable (press or don't press). The effect of all presses is a system of linear equations mod 2. You solve `Ax = b` where:

- `A` = the "toggle matrix" (which cells affect which)
- `b` = the initial grid state (flattened)
- `x` = which cells to press

Well this is what i understood when i tried to research about it 

Lets automate this using a script 

```python
import socket
import time
import numpy as np

def solve_lights_out(grid):
    n = len(grid)
    size = n * n
    A = np.zeros((size, size), dtype=int)
    for r in range(n):
        for c in range(n):
            idx = r * n + c
            A[idx][idx] = 1
            for dr, dc in [(-1,0),(1,0),(0,-1),(0,1)]:
                nr, nc = r+dr, c+dc
                if 0 <= nr < n and 0 <= nc < n:
                    A[idx][nr * n + nc] = 1
    b = np.array([cell for row in grid for cell in row], dtype=int)
    Ab = np.hstack([A, b.reshape(-1, 1)])
    pivot_cols = []
    row_idx = 0
    for col in range(size):
        pivot = None
        for r in range(row_idx, size):
            if Ab[r][col] == 1:
                pivot = r
                break
        if pivot is None:
            continue
        Ab[[row_idx, pivot]] = Ab[[pivot, row_idx]]
        pivot_cols.append(col)
        for r in range(size):
            if r != row_idx and Ab[r][col] == 1:
                Ab[r] = (Ab[r] + Ab[row_idx]) % 2
        row_idx += 1
    for r in range(row_idx, size):
        if Ab[r][-1] == 1:
            return None
    x = np.zeros(size, dtype=int)
    for i, col in enumerate(pivot_cols):
        x[col] = Ab[i][-1]
    return x.reshape(n, n)

def verify_solution(grid, solution):
    n = len(grid)
    state = [row[:] for row in grid]
    for r in range(n):
        for c in range(n):
            if solution[r][c] == 1:
                for dr, dc in [(0,0),(-1,0),(1,0),(0,-1),(0,1)]:
                    nr, nc = r+dr, c+dc
                    if 0 <= nr < n and 0 <= nc < n:
                        state[nr][nc] ^= 1
    return all(state[r][c] == 0 for r in range(n) for c in range(n))

def recv_until_marker(sock, marker, timeout=20):
    sock.settimeout(timeout)
    buf = ""
    try:
        while marker not in buf:
            ch = sock.recv(1).decode(errors='replace')
            if not ch:
                print("[WARN] Connection closed by server")
                break
            buf += ch
    except socket.timeout:
        print("[WARN] Timeout waiting for:", repr(marker))
    return buf

def parse_grid(buf):
    lines = buf.strip().split('\n')
    grid_size = None
    grid_lines = []
    reading = False
    for line in lines:
        line = line.strip()
        if line.startswith("GRID"):
            grid_size = int(line.split()[1])
            reading = True
            continue
        if reading and line and not line.startswith("AWAITING"):
            grid_lines.append(line)
        if line.startswith("AWAITING"):
            reading = False
    if grid_size is None:
        return None, None
    grid = [list(map(int, l.split())) for l in grid_lines[:grid_size]]
    return grid_size, grid

def main():
    HOST = "67.205.179.14"
    PORT = 9000

    print(f"Connecting to {HOST}:{PORT}...")
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
        s.connect((HOST, PORT))
        print("Connected!\n")

        for chamber in range(1, 21):
            print(f"\n{'='*40}")
            print(f"CHAMBER {chamber}/20")
            print('='*40)

            buf = recv_until_marker(s, "AWAITING SOLUTION\n")
            print(buf)

            grid_size, grid = parse_grid(buf)
            if grid is None:
                print("PARSE FAILED. Raw:\n", repr(buf))
                break

            print(f"Grid {grid_size}x{grid_size}:")
            for row in grid:
                print(row)

            solution = solve_lights_out(grid)
            if solution is None:
                print("No solution exists for this grid!")
                break

            if not verify_solution(grid, solution):
                print("VERIFICATION FAILED — solution is wrong!")
                break

            presses = [(r, c) for r in range(grid_size)
                               for c in range(grid_size) if solution[r][c] == 1]
            print(f"Verified OK. Sending {len(presses)} presses: {presses}")

            msg = f"PRESSES {len(presses)}\n" + "".join(f"{r} {c}\n" for r, c in presses)
            print("Sending:\n" + msg)
            s.sendall(msg.encode())

            # IMPORTANT: chamber 20 flag read BEFORE ack read
            if chamber == 20:
                time.sleep(1)
                flag_buf = ""
                s.settimeout(8)
                try:
                    while True:
                        chunk = s.recv(4096).decode(errors='replace')
                        if not chunk:
                            break
                        flag_buf += chunk
                except socket.timeout:
                    pass
                print("\n*** FLAG ***")
                print(flag_buf)
                print("repr:", repr(flag_buf))
                break

            # Chambers 1-19: consume the OK ack
            ack = recv_until_marker(s, "\n", timeout=5)
            print("Server ack:", repr(ack))

            if "flag{" in ack.lower() or "osiris{" in ack.lower() or "ctf{" in ack.lower():
                print("FLAG:", ack)
                break

if __name__ == "__main__":
    main()
```

Output:

```
Connecting to 67.205.179.14:9000...  
Connected!  
  
  
========================================  
CHAMBER 1/20  
========================================  
APERTURE SCIENCE TEST FACILITY v1.0  
GLaDOS: Welcome, test subject. Twenty chambers stand between you and the flag.  
GLaDOS: Each chamber is a grid of illuminated panels. Pressing a panel toggles it  
GLaDOS:   and each of its orthogonal neighbours. Extinguish all panels to proceed.  
GLaDOS: Send PRESSES <count> then <count> lines of '<row> <col>' (0-indexed).  
---  
CHAMBER 1/20  
GRID 6  
0 1 1 1 0 1  
1 0 0 1 1 0  
1 0 0 1 0 1  
0 0 0 0 0 1  
1 1 1 1 0 1  
1 0 1 1 0 0  
AWAITING SOLUTION  
  
Grid 6x6:  
[0, 1, 1, 1, 0, 1]  
[1, 0, 0, 1, 1, 0]  
[1, 0, 0, 1, 0, 1]  
[0, 0, 0, 0, 0, 1]  
[1, 1, 1, 1, 0, 1]  
[1, 0, 1, 1, 0, 0]  
Verified OK. Sending 17 presses: [(0, 1), (0, 2), (1, 0), (1, 1), (1, 2), (1, 5), (2, 0), (2, 2), (2, 5), (3, 0), (3, 1), (3, 4), (3, 5), (4, 0), (4, 3), (5, 0), (5, 1)]  
Sending:  
PRESSES 17  
0 1  
0 2  
1 0  
1 1  
1 2  
1 5  
2 0  
2 2  
2 5  
3 0  
3 1  
3 4  
3 5  
4 0  
4 3  
5 0  
5 1  
  
Server ack: 'OK\n'  
  
========================================  
CHAMBER 2/20  
========================================  
CHAMBER 2/20  
GRID 6  
0 0 0 1 0 0  
0 0 0 1 0 1  
0 1 0 0 0 0  
1 1 1 1 0 0  
0 0 0 0 1 1  
0 1 0 1 1 1  
AWAITING SOLUTION  
  
Grid 6x6:  
[0, 0, 0, 1, 0, 0]  
[0, 0, 0, 1, 0, 1]  
[0, 1, 0, 0, 0, 0]  
[1, 1, 1, 1, 0, 0]  
[0, 0, 0, 0, 1, 1]  
[0, 1, 0, 1, 1, 1]  
Verified OK. Sending 16 presses: [(0, 1), (0, 2), (0, 4), (1, 0), (1, 3), (1, 4), (1, 5), (2, 0), (2, 3), (2, 5), (3, 2), (3, 4), (4, 4), (5, 2), (5, 3), (5, 4)]  
Sending:  
PRESSES 16  
0 1  
0 2  
0 4  
1 0  
1 3  
1 4  
1 5  
2 0  
2 3  
2 5  
3 2  
3 4  
4 4  
5 2  
5 3  
5 4  
  
Server ack: 'OK\n'  
  
========================================  
CHAMBER 3/20  
========================================  
CHAMBER 3/20  
GRID 6  
1 0 1 1 1 0  
0 1 1 0 0 1  
0 0 1 1 1 1  
0 1 1 0 1 0  
1 1 0 1 1 0  
0 0 0 1 0 0  
AWAITING SOLUTION  
  
Grid 6x6:  
[1, 0, 1, 1, 1, 0]  
[0, 1, 1, 0, 0, 1]  
[0, 0, 1, 1, 1, 1]  
[0, 1, 1, 0, 1, 0]  
[1, 1, 0, 1, 1, 0]  
[0, 0, 0, 1, 0, 0]  
Verified OK. Sending 18 presses: [(0, 0), (0, 2), (1, 4), (2, 0), (2, 1), (2, 3), (2, 4), (3, 2), (3, 3), (4, 0), (4, 1), (4, 2), (4, 3), (4, 4), (5, 0), (5, 3), (5, 4), (5, 5)]  
Sending:  
PRESSES 18  
0 0  
0 2  
1 4  
2 0  
2 1  
2 3  
2 4  
3 2  
3 3  
4 0  
4 1  
4 2  
4 3  
4 4  
5 0  
5 3  
5 4  
5 5  
  
Server ack: 'OK\n'  
  
========================================  
CHAMBER 4/20  
========================================  
CHAMBER 4/20  
GRID 5  
0 1 0 1 1  
1 0 1 0 1  
1 0 0 1 1  
1 0 0 1 1  
1 1 1 0 0  
AWAITING SOLUTION  
  
Grid 5x5:  
[0, 1, 0, 1, 1]  
[1, 0, 1, 0, 1]  
[1, 0, 0, 1, 1]  
[1, 0, 0, 1, 1]  
[1, 1, 1, 0, 0]  
Verified OK. Sending 10 presses: [(0, 0), (0, 2), (1, 0), (1, 1), (1, 2), (1, 4), (2, 1), (3, 0), (3, 3), (4, 2)]  
Sending:  
PRESSES 10  
0 0  
0 2  
1 0  
1 1  
1 2  
1 4  
2 1  
3 0  
3 3  
4 2  
  
Server ack: 'OK\n'  
  
========================================  
CHAMBER 5/20  
========================================  
CHAMBER 5/20  
GRID 5  
0 1 1 1 1  
0 1 0 1 0  
0 1 1 1 1  
0 1 1 0 1  
1 1 1 0 0  
AWAITING SOLUTION  
  
Grid 5x5:  
[0, 1, 1, 1, 1]  
[0, 1, 0, 1, 0]  
[0, 1, 1, 1, 1]  
[0, 1, 1, 0, 1]  
[1, 1, 1, 0, 0]  
Verified OK. Sending 12 presses: [(0, 1), (0, 3), (0, 4), (1, 0), (1, 2), (1, 3), (1, 4), (2, 0), (2, 3), (2, 4), (3, 2), (4, 0)]  
Sending:  
PRESSES 12  
0 1  
0 3  
0 4  
1 0  
1 2  
1 3  
1 4  
2 0  
2 3  
2 4  
3 2  
4 0  
  
Server ack: 'OK\n'  
  
========================================  
CHAMBER 6/20  
========================================  
CHAMBER 6/20  
GRID 7  
1 1 0 0 0 0 1  
0 1 0 0 0 1 1  
1 0 1 1 0 0 1  
0 1 1 1 0 1 1  
1 0 1 1 1 1 1  
1 0 1 1 0 0 0  
1 1 1 0 1 1 1  
AWAITING SOLUTION  
  
Grid 7x7:  
[1, 1, 0, 0, 0, 0, 1]  
[0, 1, 0, 0, 0, 1, 1]  
[1, 0, 1, 1, 0, 0, 1]  
[0, 1, 1, 1, 0, 1, 1]  
[1, 0, 1, 1, 1, 1, 1]  
[1, 0, 1, 1, 0, 0, 0]  
[1, 1, 1, 0, 1, 1, 1]  
Verified OK. Sending 26 presses: [(0, 1), (0, 2), (0, 3), (0, 5), (1, 1), (1, 2), (1, 5), (2, 0), (2, 2), (2, 4), (2, 5), (3, 1), (3, 2), (3, 3), (3, 5), (4, 1), (4, 2), (4, 3), (4, 4), (4, 5), (5, 1), (5, 2), (5, 3), (6, 1), (6, 2), (6,  
5)]  
Sending:  
PRESSES 26  
0 1  
0 2  
0 3  
0 5  
1 1  
1 2  
1 5  
2 0  
2 2  
2 4  
2 5  
3 1  
3 2  
3 3  
3 5  
4 1  
4 2  
4 3  
4 4  
4 5  
5 1  
5 2  
5 3  
6 1  
6 2  
6 5  
  
Server ack: 'OK\n'  
  
========================================  
CHAMBER 7/20  
========================================  
CHAMBER 7/20  
GRID 7  
1 0 0 0 1 1 0  
0 1 0 0 0 1 1  
0 1 0 1 0 1 0  
0 1 0 1 1 1 1  
0 1 0 0 1 0 0  
0 0 0 1 0 1 1  
0 1 0 1 1 1 0  
AWAITING SOLUTION  
  
Grid 7x7:  
[1, 0, 0, 0, 1, 1, 0]  
[0, 1, 0, 0, 0, 1, 1]  
[0, 1, 0, 1, 0, 1, 0]  
[0, 1, 0, 1, 1, 1, 1]  
[0, 1, 0, 0, 1, 0, 0]  
[0, 0, 0, 1, 0, 1, 1]  
[0, 1, 0, 1, 1, 1, 0]  
Verified OK. Sending 20 presses: [(0, 0), (0, 2), (0, 4), (0, 5), (0, 6), (1, 2), (1, 4), (2, 0), (2, 5), (3, 0), (3, 2), (3, 3), (3, 6), (4, 1), (4, 3), (4, 5), (5, 2), (5, 4), (5, 5), (6, 2)]  
Sending:  
PRESSES 20  
0 0  
0 2  
0 4  
0 5  
0 6  
1 2  
1 4  
2 0  
2 5  
3 0  
3 2  
3 3  
3 6  
4 1  
4 3  
4 5  
5 2  
5 4  
5 5  
6 2  
  
Server ack: 'OK\n'  
  
========================================  
CHAMBER 8/20  
========================================  
CHAMBER 8/20  
GRID 5  
1 1 0 1 0  
0 1 0 1 0  
0 0 0 0 1  
0 1 0 0 0  
1 1 1 1 1  
AWAITING SOLUTION  
  
Grid 5x5:  
[1, 1, 0, 1, 0]  
[0, 1, 0, 1, 0]  
[0, 0, 0, 0, 1]  
[0, 1, 0, 0, 0]  
[1, 1, 1, 1, 1]  
Verified OK. Sending 15 presses: [(0, 0), (0, 1), (0, 2), (0, 3), (1, 0), (1, 2), (1, 3), (1, 4), (2, 2), (2, 3), (3, 0), (3, 1), (3, 2), (3, 3), (3, 4)]  
Sending:  
PRESSES 15  
0 0  
0 1  
0 2  
0 3  
1 0  
1 2  
1 3  
1 4  
2 2  
2 3  
3 0  
3 1  
3 2  
3 3  
3 4  
  
Server ack: 'OK\n'  
  
========================================  
CHAMBER 9/20  
========================================  
CHAMBER 9/20  
GRID 6  
1 0 0 1 0 1  
0 0 0 1 1 1  
1 1 1 1 0 0  
1 1 0 1 0 1  
1 0 1 0 1 0  
0 1 0 0 1 1  
AWAITING SOLUTION  
  
Grid 6x6:  
[1, 0, 0, 1, 0, 1]  
[0, 0, 0, 1, 1, 1]  
[1, 1, 1, 1, 0, 0]  
[1, 1, 0, 1, 0, 1]  
[1, 0, 1, 0, 1, 0]  
[0, 1, 0, 0, 1, 1]  
Verified OK. Sending 16 presses: [(0, 0), (0, 1), (0, 3), (0, 5), (1, 0), (2, 4), (3, 1), (3, 2), (3, 4), (3, 5), (4, 1), (4, 3), (4, 4), (4, 5), (5, 4), (5, 5)]  
Sending:  
PRESSES 16  
0 0  
0 1  
0 3  
0 5  
1 0  
2 4  
3 1  
3 2  
3 4  
3 5  
4 1  
4 3  
4 4  
4 5  
5 4  
5 5  
  
Server ack: 'OK\n'  
  
========================================  
CHAMBER 10/20  
========================================  
CHAMBER 10/20  
GRID 5  
0 0 0 1 1  
1 0 0 1 1  
0 1 0 0 1  
0 0 1 0 1  
1 1 1 1 1  
AWAITING SOLUTION  
  
Grid 5x5:  
[0, 0, 0, 1, 1]  
[1, 0, 0, 1, 1]  
[0, 1, 0, 0, 1]  
[0, 0, 1, 0, 1]  
[1, 1, 1, 1, 1]  
Verified OK. Sending 12 presses: [(0, 1), (0, 2), (0, 3), (0, 4), (1, 0), (1, 2), (1, 4), (2, 1), (2, 4), (3, 3), (3, 4), (4, 1)]  
Sending:  
PRESSES 12  
0 1  
0 2  
0 3  
0 4  
1 0  
1 2  
1 4  
2 1  
2 4  
3 3  
3 4  
4 1  
  
Server ack: 'OK\n'  
  
========================================  
CHAMBER 11/20  
========================================  
CHAMBER 11/20  
GRID 7  
0 0 0 1 0 0 1  
0 0 1 1 0 0 0  
1 1 1 1 0 0 0  
1 1 0 0 0 0 1  
1 1 0 0 1 1 0  
1 0 1 1 1 0 0  
0 1 0 1 0 0 1  
AWAITING SOLUTION  
  
Grid 7x7:  
[0, 0, 0, 1, 0, 0, 1]  
[0, 0, 1, 1, 0, 0, 0]  
[1, 1, 1, 1, 0, 0, 0]  
[1, 1, 0, 0, 0, 0, 1]  
[1, 1, 0, 0, 1, 1, 0]  
[1, 0, 1, 1, 1, 0, 0]  
[0, 1, 0, 1, 0, 0, 1]  
Verified OK. Sending 28 presses: [(0, 0), (0, 2), (0, 3), (0, 5), (0, 6), (1, 0), (1, 3), (1, 6), (2, 1), (2, 2), (2, 3), (2, 4), (3, 0), (3, 1), (3, 3), (3, 5), (3, 6), (4, 0), (4, 2), (4, 4), (4, 6), (5, 0), (5, 2), (5, 3), (6, 0), (6,  
3), (6, 4), (6, 6)]  
Sending:  
PRESSES 28  
0 0  
0 2  
0 3  
0 5  
0 6  
1 0  
1 3  
1 6  
2 1  
2 2  
2 3  
2 4  
3 0  
3 1  
3 3  
3 5  
3 6  
4 0  
4 2  
4 4  
4 6  
5 0  
5 2  
5 3  
6 0  
6 3  
6 4  
6 6  
  
Server ack: 'OK\n'  
  
========================================  
CHAMBER 12/20  
========================================  
CHAMBER 12/20  
GRID 6  
1 1 1 0 0 1  
0 1 1 1 0 0  
0 1 0 0 0 0  
0 1 1 0 1 1  
1 0 0 0 1 0  
0 1 1 1 0 1  
AWAITING SOLUTION  
  
Grid 6x6:  
[1, 1, 1, 0, 0, 1]  
[0, 1, 1, 1, 0, 0]  
[0, 1, 0, 0, 0, 0]  
[0, 1, 1, 0, 1, 1]  
[1, 0, 0, 0, 1, 0]  
[0, 1, 1, 1, 0, 1]  
Verified OK. Sending 21 presses: [(0, 0), (0, 2), (0, 4), (1, 1), (1, 4), (2, 2), (2, 5), (3, 1), (3, 2), (3, 3), (3, 5), (4, 0), (4, 1), (4, 2), (4, 4), (4, 5), (5, 0), (5, 2), (5, 3), (5, 4), (5, 5)]  
Sending:  
PRESSES 21  
0 0  
0 2  
0 4  
1 1  
1 4  
2 2  
2 5  
3 1  
3 2  
3 3  
3 5  
4 0  
4 1  
4 2  
4 4  
4 5  
5 0  
5 2  
5 3  
5 4  
5 5  
  
Server ack: 'OK\n'  
  
========================================  
CHAMBER 13/20  
========================================  
CHAMBER 13/20  
GRID 6  
1 0 0 1 0 0  
1 0 1 0 0 1  
0 0 1 1 1 0  
1 0 0 1 1 0  
1 0 0 1 0 0  
0 0 1 1 0 1  
AWAITING SOLUTION  
  
Grid 6x6:  
[1, 0, 0, 1, 0, 0]  
[1, 0, 1, 0, 0, 1]  
[0, 0, 1, 1, 1, 0]  
[1, 0, 0, 1, 1, 0]  
[1, 0, 0, 1, 0, 0]  
[0, 0, 1, 1, 0, 1]  
Verified OK. Sending 13 presses: [(0, 1), (0, 2), (0, 3), (1, 2), (1, 3), (1, 4), (2, 0), (3, 0), (3, 1), (4, 2), (4, 3), (4, 4), (5, 5)]  
Sending:  
PRESSES 13  
0 1  
0 2  
0 3  
1 2  
1 3  
1 4  
2 0  
3 0  
3 1  
4 2  
4 3  
4 4  
5 5  
  
Server ack: 'OK\n'  
  
========================================  
CHAMBER 14/20  
========================================  
CHAMBER 14/20  
GRID 6  
1 0 0 0 1 0  
1 1 0 0 0 1  
1 1 1 1 1 1  
0 0 0 1 1 0  
0 0 1 0 1 0  
0 1 1 0 1 0  
AWAITING SOLUTION  
  
Grid 6x6:  
[1, 0, 0, 0, 1, 0]  
[1, 1, 0, 0, 0, 1]  
[1, 1, 1, 1, 1, 1]  
[0, 0, 0, 1, 1, 0]  
[0, 0, 1, 0, 1, 0]  
[0, 1, 1, 0, 1, 0]  
Verified OK. Sending 23 presses: [(0, 2), (1, 0), (1, 1), (1, 2), (1, 3), (1, 4), (2, 0), (2, 3), (3, 0), (3, 1), (3, 2), (3, 3), (3, 4), (3, 5), (4, 0), (4, 1), (4, 2), (4, 3), (5, 0), (5, 2), (5, 3), (5, 4), (5, 5)]  
Sending:  
PRESSES 23  
0 2  
1 0  
1 1  
1 2  
1 3  
1 4  
2 0  
2 3  
3 0  
3 1  
3 2  
3 3  
3 4  
3 5  
4 0  
4 1  
4 2  
4 3  
5 0  
5 2  
5 3  
5 4  
5 5  
  
Server ack: 'OK\n'  
  
========================================  
CHAMBER 15/20  
========================================  
CHAMBER 15/20  
GRID 7  
1 0 1 1 1 1 1  
1 0 0 0 0 1 1  
0 1 1 0 1 1 0  
1 0 1 0 0 0 1  
0 0 1 1 0 0 1  
1 0 0 1 0 0 1  
0 1 0 0 0 1 0  
AWAITING SOLUTION  
  
Grid 7x7:  
[1, 0, 1, 1, 1, 1, 1]  
[1, 0, 0, 0, 0, 1, 1]  
[0, 1, 1, 0, 1, 1, 0]  
[1, 0, 1, 0, 0, 0, 1]  
[0, 0, 1, 1, 0, 0, 1]  
[1, 0, 0, 1, 0, 0, 1]  
[0, 1, 0, 0, 0, 1, 0]  
Verified OK. Sending 22 presses: [(0, 0), (0, 2), (0, 3), (0, 4), (0, 5), (1, 2), (1, 5), (2, 1), (2, 5), (3, 0), (3, 2), (3, 5), (3, 6), (4, 1), (4, 3), (4, 4), (4, 5), (4, 6), (5, 1), (5, 3), (5, 4), (6, 4)]  
Sending:  
PRESSES 22  
0 0  
0 2  
0 3  
0 4  
0 5  
1 2  
1 5  
2 1  
2 5  
3 0  
3 2  
3 5  
3 6  
4 1  
4 3  
4 4  
4 5  
4 6  
5 1  
5 3  
5 4  
6 4  
  
Server ack: 'OK\n'  
  
========================================  
CHAMBER 16/20  
========================================  
CHAMBER 16/20  
GRID 5  
0 1 1 1 0  
0 1 0 0 0  
1 0 0 0 0  
0 1 1 1 0  
0 0 0 1 0  
AWAITING SOLUTION  
  
Grid 5x5:  
[0, 1, 1, 1, 0]  
[0, 1, 0, 0, 0]  
[1, 0, 0, 0, 0]  
[0, 1, 1, 1, 0]  
[0, 0, 0, 1, 0]  
Verified OK. Sending 9 presses: [(0, 0), (0, 2), (1, 0), (1, 1), (2, 0), (2, 1), (3, 1), (3, 2), (4, 2)]  
Sending:  
PRESSES 9  
0 0  
0 2  
1 0  
1 1  
2 0  
2 1  
3 1  
3 2  
4 2  
  
Server ack: 'OK\n'  
  
========================================  
CHAMBER 17/20  
========================================  
CHAMBER 17/20  
GRID 6  
1 0 1 1 1 0  
1 1 0 0 1 0  
1 0 0 0 1 1  
0 1 0 1 0 1  
0 1 0 0 1 0  
0 1 0 1 1 0  
AWAITING SOLUTION  
  
Grid 6x6:  
[1, 0, 1, 1, 1, 0]  
[1, 1, 0, 0, 1, 0]  
[1, 0, 0, 0, 1, 1]  
[0, 1, 0, 1, 0, 1]  
[0, 1, 0, 0, 1, 0]  
[0, 1, 0, 1, 1, 0]  
Verified OK. Sending 20 presses: [(1, 0), (1, 2), (1, 3), (1, 4), (2, 1), (2, 3), (2, 4), (2, 5), (3, 0), (3, 1), (3, 2), (3, 3), (3, 4), (3, 5), (4, 1), (4, 2), (4, 3), (5, 3), (5, 4), (5, 5)]  
Sending:  
PRESSES 20  
1 0  
1 2  
1 3  
1 4  
2 1  
2 3  
2 4  
2 5  
3 0  
3 1  
3 2  
3 3  
3 4  
3 5  
4 1  
4 2  
4 3  
5 3  
5 4  
5 5  
  
Server ack: 'OK\n'  
  
========================================  
CHAMBER 18/20  
========================================  
CHAMBER 18/20  
GRID 6  
1 0 1 0 1 0  
0 1 0 0 0 0  
0 0 0 1 1 1  
1 1 1 1 1 1  
1 1 0 0 1 1  
0 1 1 0 0 0  
AWAITING SOLUTION  
  
Grid 6x6:  
[1, 0, 1, 0, 1, 0]  
[0, 1, 0, 0, 0, 0]  
[0, 0, 0, 1, 1, 1]  
[1, 1, 1, 1, 1, 1]  
[1, 1, 0, 0, 1, 1]  
[0, 1, 1, 0, 0, 0]  
Verified OK. Sending 17 presses: [(0, 1), (0, 4), (1, 1), (1, 3), (1, 5), (2, 0), (2, 1), (2, 3), (2, 4), (2, 5), (3, 1), (4, 0), (4, 1), (5, 0), (5, 2), (5, 4), (5, 5)]  
Sending:  
PRESSES 17  
0 1  
0 4  
1 1  
1 3  
1 5  
2 0  
2 1  
2 3  
2 4  
2 5  
3 1  
4 0  
4 1  
5 0  
5 2  
5 4  
5 5  
  
Server ack: 'OK\n'  
  
========================================  
CHAMBER 19/20  
========================================  
CHAMBER 19/20  
GRID 6  
0 1 0 1 0 1  
1 0 1 1 1 1  
0 0 0 0 1 0  
1 1 1 0 1 1  
1 0 0 1 1 0  
0 1 1 0 1 0  
AWAITING SOLUTION  
  
Grid 6x6:  
[0, 1, 0, 1, 0, 1]  
[1, 0, 1, 1, 1, 1]  
[0, 0, 0, 0, 1, 0]  
[1, 1, 1, 0, 1, 1]  
[1, 0, 0, 1, 1, 0]  
[0, 1, 1, 0, 1, 0]  
Verified OK. Sending 22 presses: [(0, 5), (1, 1), (1, 3), (1, 4), (2, 1), (2, 2), (2, 3), (2, 4), (2, 5), (3, 0), (3, 1), (3, 2), (3, 4), (4, 0), (4, 1), (4, 3), (4, 4), (4, 5), (5, 1), (5, 2), (5, 3), (5, 4)]  
Sending:  
PRESSES 22  
0 5  
1 1  
1 3  
1 4  
2 1  
2 2  
2 3  
2 4  
2 5  
3 0  
3 1  
3 2  
3 4  
4 0  
4 1  
4 3  
4 4  
4 5  
5 1  
5 2  
5 3  
5 4  
  
Server ack: 'OK\n'  
  
========================================  
CHAMBER 20/20  
========================================  
CHAMBER 20/20  
GRID 7  
0 0 0 1 1 0 1  
1 1 0 1 0 1 1  
0 1 0 1 1 1 1  
0 0 0 0 1 1 0  
0 1 1 1 1 0 0  
0 1 1 0 0 0 1  
1 0 1 0 1 1 1  
AWAITING SOLUTION  
  
Grid 7x7:  
[0, 0, 0, 1, 1, 0, 1]  
[1, 1, 0, 1, 0, 1, 1]  
[0, 1, 0, 1, 1, 1, 1]  
[0, 0, 0, 0, 1, 1, 0]  
[0, 1, 1, 1, 1, 0, 0]  
[0, 1, 1, 0, 0, 0, 1]  
[1, 0, 1, 0, 1, 1, 1]  
Verified OK. Sending 26 presses: [(0, 0), (0, 2), (0, 5), (0, 6), (1, 0), (1, 2), (1, 6), (2, 0), (2, 1), (2, 5), (2, 6), (3, 0), (3, 1), (3, 3), (3, 5), (4, 0), (4, 1), (4, 3), (4, 4), (4, 5), (5, 0), (5, 2), (5, 5), (5, 6), (6, 5), (6,  
6)]  
Sending:  
PRESSES 26  
0 0  
0 2  
0 5  
0 6  
1 0  
1 2  
1 6  
2 0  
2 1  
2 5  
2 6  
3 0  
3 1  
3 3  
3 5  
4 0  
4 1  
4 3  
4 4  
4 5  
5 0  
5 2  
5 5  
5 6  
6 5  
6 6  
  
  
*** FLAG ***  
OK  
FLAG bitctf{{gl4d0s_s4ys_y0u_p4ss3d_4ll_ch4mb3rs}}  
GLaDOS: This was a triumph. I'm making a note here: HUGE SUCCESS.  
  
repr: "OK\nFLAG bitctf{{gl4d0s_s4ys_y0u_p4ss3d_4ll_ch4mb3rs}}\nGLaDOS: This was a triumph. I'm making a note here: HUGE SUCCESS.\n"
```

There it is we got the flag

```
bitctf{{gl4d0s_s4ys_y0u_p4ss3d_4ll_ch4mb3rs}} 
```

