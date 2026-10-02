 @Test
    fun findPageReturnsSingleClient() {

        val repository = repository()

        insertClients(1)

        val result = repository.findPage(
            ownerId = "test-owner",
            yearMonth = "2026-09",
            status = ClientType.RECEIVABLE,
            size = 10,
            cursor = null
        )

        assertEquals(1, result.items.size)
        assertEquals("client-000", result.items[0].clientId)
        assertFalse(result.hasNext)
        assertNull(result.nextCursor)
    }
