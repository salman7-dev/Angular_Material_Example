fun save(
        history: ClientMonthlyHistory,
        bucketId: String
    ) {
        val bucketDocument = getBucketDocument(
            history.yearMonth,
            bucketId
        )

        val clientDocument = getClientHistoryDocument(
            history.yearMonth,
            bucketId,
            history.clientId
        )

        firestore.runTransaction { transaction ->

            val bucketSnapshot = transaction
                .get(bucketDocument)
                .get()

            if (!bucketSnapshot.exists()) {

                val bucket = ClientHistoryBucket(
                    capacity = CAPACITY,
                    size = 1
                )

                transaction.set(bucketDocument, bucket)
                transaction.set(clientDocument, history)

                return@runTransaction
            }

            val bucket = bucketSnapshot
                .toObject(ClientHistoryBucket::class.java)
                ?: ClientHistoryBucket(capacity = CAPACITY)

            val clientSnapshot = transaction
                .get(clientDocument)
                .get()

            if (!clientSnapshot.exists()) {

                if (bucket.size >= bucket.capacity) {
                    throw IllegalStateException(
                        "Bucket $bucketId is full"
                    )
                }

                transaction.update(
                    bucketDocument,
                    "size",
                    bucket.size + 1
                )
            }

            transaction.set(clientDocument, history)
        }.get()
    }
