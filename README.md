PS D:\New folder\client-ledger-backend> Get-ChildItem .\src\test\kotlin -Recurse -Filter *.kt |
>>     Select-String -Pattern "org\.springframework\.boot\.test\.autoconfigure\.web\.servlet|com\.fasterxml\.jackson|io\.mockk"

src\test\kotlin\com\clientledger\core\integration\client\ClientControllerIntegrationTest.kt:7:import com.fasterxml.jackson.databind.ObjectMapper
src\test\kotlin\com\clientledger\core\integration\client\ClientControllerIntegrationTest.kt:14:import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc
src\test\kotlin\com\clientledger\core\integration\owner\OwnerControllerIntegrationTest.kt:7:import com.fasterxml.jackson.databind.ObjectMapper
src\test\kotlin\com\clientledger\core\integration\owner\OwnerControllerIntegrationTest.kt:14:import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc
src\test\kotlin\com\clientledger\core\integration\security\FirebaseAuthEmulatorClient.kt:3:import com.fasterxml.jackson.databind.JsonNode
src\test\kotlin\com\clientledger\core\integration\security\FirebaseAuthEmulatorClient.kt:4:import com.fasterxml.jackson.databind.ObjectMapper
src\test\kotlin\com\clientledger\core\integration\security\FirebaseSecurityIntegrationTest.kt:6:import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc
src\test\kotlin\com\clientledger\core\integration\security\LocalAuthDisabledControllerIntegrationTest.kt:5:import com.fasterxml.jackson.databind.ObjectMapper
src\test\kotlin\com\clientledger\core\integration\security\LocalAuthDisabledControllerIntegrationTest.kt:11:import 
org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc


PS D:\New folder\client-ledger-backend>
