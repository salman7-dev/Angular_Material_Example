PS D:\New folder\client-ledger-codespace-main> curl.exe http://127.0.0.1:8080
Ok
PS D:\New folder\client-ledger-codespace-main> netstat -ano | findstr :8080
  TCP    127.0.0.1:8080         0.0.0.0:0              LISTENING       14612
  TCP    127.0.0.1:8080         127.0.0.1:63223        ESTABLISHED     14612
  TCP    127.0.0.1:8080         127.0.0.1:63225        ESTABLISHED     14612
  TCP    127.0.0.1:63223        127.0.0.1:8080         ESTABLISHED     19216
  TCP    127.0.0.1:63225        127.0.0.1:8080         ESTABLISHED     14612
  TCP    127.0.0.1:64257        127.0.0.1:8080         TIME_WAIT       0
PS D:\New folder\client-ledger-codespace-main> 
