application-test.yml
client-ledger:

  auth:
    enabled: false
    local-owner-id: local-owner
    mode: EMULATOR
    emulator-host: "127.0.0.1:9099"

  history:
    editable-months: 2
    bucket-capacity: 100

  summary:
    index-bucket-capacity: 300

  firestore:
    project-id: "client-ledger-dashboard"
    host: "127.0.0.1:8080"
    emulator-host: "127.0.0.1:8080"

