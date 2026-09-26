PS D:\New folder\client-ledger-codespace-main> Get-ChildItem -Recurse -File app\src | Select-String "demo-no-project"     

app\src\main\resources\application.yml:10:    project-id: "demo-no-project"
app\src\test\kotlin\com\clientledger\core\integration\client\ClientCreateIntegrationTest.kt:45:                .setProjectId("demo-no-project")
app\src\test\kotlin\com\clientledger\core\integration\client\SummaryIntegrationTest.kt:41:                .setProjectId("demo-no-project")
app\src\test\kotlin\com\clientledger\core\integration\client\security\FirebaseAuthEmulatorClient.kt:11:    private val projectId: String = "demo-no-project"
app\src\test\kotlin\com\clientledger\core\repository\client\ClientRepositoryTest.kt:27:                .setProjectId("demo-no-project")
app\src\test\kotlin\com\clientledger\core\repository\history\ClientHistoryBucketRepositoryTest.kt:23:                .setProjectId("demo-no-project")
app\src\test\kotlin\com\clientledger\core\repository\history\ClientHistoryRepositoryTest.kt:26:                .setProjectId("demo-no-project")
app\src\test\kotlin\com\clientledger\core\repository\summary\SummaryClientIndexRepositoryTest.kt:25:                .setProjectId("demo-no-project")
app\src\test\kotlin\com\clientledger\core\service\client\ClientServiceTest.kt:49:                .setProjectId("demo-no-project")
app\src\test\kotlin\com\clientledger\core\service\history\ClientHistoryServiceTest.kt:39:                .setProjectId("demo-no-project")
app\src\test\kotlin\com\clientledger\core\service\summary\GlobalSummaryRepositoryTest.kt:22:                .setProjectId("demo-no-project")
app\src\test\kotlin\com\clientledger\core\service\summary\SummaryClientIndexBucketRepositoryTest.kt:33:                .setProjectId("demo-no-project")
app\src\test\resources\application-test.yml:17:    project-id: "demo-no-project"


PS D:\New folder\client-ledger-codespace-main> 
