package com.clientledger.core.service

import com.clientledger.core.repository.client.ClientRepository
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Test
import org.mockito.kotlin.mock
import org.mockito.kotlin.whenever

class ClientHistoryBucketServiceTest {

    private val clientRepository =
        mock<ClientRepository>()

    private val service =
        ClientHistoryBucketService(clientRepository)

    @Test
    fun allocatesFirstBucket() {
        whenever(clientRepository.count())
            .thenReturn(0)

        assertEquals(
            "bucket_000",
            service.allocateBucket()
        )
    }

    @Test
    fun keepsFiftyClientsInSameBucket() {
        whenever(clientRepository.count())
            .thenReturn(49)

        assertEquals(
            "bucket_000",
            service.allocateBucket()
        )
    }

    @Test
    fun allocatesNextBucketAfterFiftyClients() {
        whenever(clientRepository.count())
            .thenReturn(50)

        assertEquals(
            "bucket_001",
            service.allocateBucket()
        )
    }

    @Test
    fun allocatesThirdBucketAfterHundredClients() {
        whenever(clientRepository.count())
            .thenReturn(100)

        assertEquals(
            "bucket_002",
            service.allocateBucket()
        )
    }
}
