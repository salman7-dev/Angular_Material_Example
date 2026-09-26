PS D:\New folder\client-ledger-codespace-main> Get-ChildItem -Recurse -File app\src\test | Select-String "client-ledger-dashboard"

app\src\test\kotlin\com\clientledger\core\integration\client\ClientCreateIntegrationTest.kt:45:                .setProjectId("client-ledger-dashboard")
app\src\test\kotlin\com\clientledger\core\integration\client\SummaryIntegrationTest.kt:41:                .setProjectId("client-ledger-dashboard")
app\src\test\kotlin\com\clientledger\core\repository\client\ClientRepositoryTest.kt:27:                .setProjectId("client-ledger-dashboard")
app\src\test\kotlin\com\clientledger\core\repository\history\ClientHistoryBucketRepositoryTest.kt:23:                .setProjectId("client-ledger-dashboard")
app\src\test\kotlin\com\clientledger\core\repository\history\ClientHistoryRepositoryTest.kt:26:                .setProjectId("client-ledger-dashboard")
app\src\test\kotlin\com\clientledger\core\repository\summary\SummaryClientIndexRepositoryTest.kt:25:                .setProjectId("client-ledger-dashboard")
app\src\test\kotlin\com\clientledger\core\service\client\ClientServiceTest.kt:49:                .setProjectId("client-ledger-dashboard")
app\src\test\kotlin\com\clientledger\core\service\history\ClientHistoryServiceTest.kt:39:                .setProjectId("client-ledger-dashboard")
app\src\test\kotlin\com\clientledger\core\service\summary\GlobalSummaryRepositoryTest.kt:22:                .setProjectId("client-ledger-dashboard")
app\src\test\kotlin\com\clientledger\core\service\summary\SummaryClientIndexBucketRepositoryTest.kt:33:                .setProjectId("client-ledger-dashboard")


PS D:\New folder\client-ledger-codespace-main> 
