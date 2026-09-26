PS D:\New folder\client-ledger-codespace-main> Get-Content firebase.json
{
  "firestore": {
    "rules": "firestore.rules"
  },
  "emulators": {
    "firestore": {
      "port": 8080
    },
    "auth": {
      "port": 9099
    },
    "ui": {
      "enabled": true,
      "port": 4000
    }
  }
}
PS D:\New folder\client-ledger-codespace-main> $env:FIREBASE_AUTH_EMULATOR_HOST
127.0.0.1:9099
PS D:\New folder\client-ledger-codespace-main> 
