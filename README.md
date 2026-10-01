package com.clientledger.core.repository.monthlyrollover

import com.clientledger.core.domain.MonthlyRolloverState
import com.google.cloud.Timestamp
import com.google.cloud.firestore.DocumentReference
import com.google.cloud.firestore.DocumentSnapshot
import com.google.cloud.firestore.Firestore
import org.springframework.stereotype.Repository
import java.time.Instant

@Repository
class MonthlyRolloverRepository(
    private val firestore: Firestore
) {

    private fun rolloverDocument(
        ownerId: String,
        yearMonth: String
    ): DocumentReference {

        validateOwnerId(ownerId)
        validateYearMonth(yearMonth)

        return firestore
            .collection("owners")
            .document(ownerId)
            .collection("monthly_rollover")
            .document(yearMonth)
    }

    private fun retryOwnerDocument(
        ownerId: String,
        yearMonth: String
    ): DocumentReference {

        validateOwnerId(ownerId)
        validateYearMonth(yearMonth)

        return firestore
            .collection("system")
            .document("monthly_rollover_retry")
            .collection(yearMonth)
            .document(ownerId)
    }

    fun find(
        ownerId: String,
        yearMonth: String
    ): MonthlyRolloverState? {

        val snapshot =
            rolloverDocument(
                ownerId = ownerId,
                yearMonth = yearMonth
            )
                .get()
                .get()

        if (!snapshot.exists()) {
            return null
        }

        return toState(snapshot)
    }

    fun createPending(
        ownerId: String,
        yearMonth: String
    ): MonthlyRolloverState {

        val rolloverReference =
            rolloverDocument(
                ownerId = ownerId,
                yearMonth = yearMonth
            )

        return firestore
            .runTransaction { transaction ->

                val existing =
                    transaction
                        .get(rolloverReference)
                        .get()

                require(!existing.exists()) {
                    "Monthly rollover state already exists: $ownerId/$yearMonth"
                }

                val state =
                    MonthlyRolloverState(
                        ownerId = ownerId,
                        yearMonth = yearMonth,
                        status = MonthlyRolloverState.Status.PENDING
                    )

                transaction.set(
                    rolloverReference,
                    toDocument(state)
                )

                state
            }
            .get()
    }

    fun markRunning(
        ownerId: String,
        yearMonth: String,
        startedAt: Instant
    ): MonthlyRolloverState {

        val rolloverReference =
            rolloverDocument(
                ownerId = ownerId,
                yearMonth = yearMonth
            )

        val retryReference =
            retryOwnerDocument(
                ownerId = ownerId,
                yearMonth = yearMonth
            )

        return firestore
            .runTransaction { transaction ->

                val rolloverSnapshot =
                    transaction
                        .get(rolloverReference)
                        .get()

                require(rolloverSnapshot.exists()) {
                    "Monthly rollover state does not exist: $ownerId/$yearMonth"
                }

                val current =
                    toState(rolloverSnapshot)

                require(
                    current.status !=
                            MonthlyRolloverState.Status.COMPLETED
                ) {
                    "Monthly rollover is already completed: $ownerId/$yearMonth"
                }

                val updated =
                    current.copy(
                        status = MonthlyRolloverState.Status.RUNNING,
                        attempt = current.attempt + 1,
                        startedAt = startedAt,
                        completedAt = null,
                        failedAt = null,
                        error = null
                    )

                transaction.set(
                    rolloverReference,
                    toDocument(updated)
                )

                /*
                 * If this is a retry of a failed owner,
                 * remove the retry lookup entry atomically.
                 */
                transaction.delete(retryReference)

                updated
            }
            .get()
    }

    fun markCompleted(
        ownerId: String,
        yearMonth: String,
        completedAt: Instant
    ): MonthlyRolloverState {

        val rolloverReference =
            rolloverDocument(
                ownerId = ownerId,
                yearMonth = yearMonth
            )

        val retryReference =
            retryOwnerDocument(
                ownerId = ownerId,
                yearMonth = yearMonth
            )

        return firestore
            .runTransaction { transaction ->

                val rolloverSnapshot =
                    transaction
                        .get(rolloverReference)
                        .get()

                require(rolloverSnapshot.exists()) {
                    "Monthly rollover state does not exist: $ownerId/$yearMonth"
                }

                val current =
                    toState(rolloverSnapshot)

                require(
                    current.status ==
                            MonthlyRolloverState.Status.RUNNING
                ) {
                    "Monthly rollover must be RUNNING before completion: $ownerId/$yearMonth"
                }

                val updated =
                    current.copy(
                        status = MonthlyRolloverState.Status.COMPLETED,
                        completedAt = completedAt,
                        failedAt = null,
                        error = null
                    )

                transaction.set(
                    rolloverReference,
                    toDocument(updated)
                )

                /*
                 * Successful completion must remove the owner
                 * from the failed-owner retry index.
                 */
                transaction.delete(retryReference)

                updated
            }
            .get()
    }

    fun markFailed(
        ownerId: String,
        yearMonth: String,
        failedAt: Instant,
        error: String?
    ): MonthlyRolloverState {

        val rolloverReference =
            rolloverDocument(
                ownerId = ownerId,
                yearMonth = yearMonth
            )

        val retryReference =
            retryOwnerDocument(
                ownerId = ownerId,
                yearMonth = yearMonth
            )

        return firestore
            .runTransaction { transaction ->

                val rolloverSnapshot =
                    transaction
                        .get(rolloverReference)
                        .get()

                require(rolloverSnapshot.exists()) {
                    "Monthly rollover state does not exist: $ownerId/$yearMonth"
                }

                val current =
                    toState(rolloverSnapshot)

                require(
                    current.status ==
                            MonthlyRolloverState.Status.RUNNING
                ) {
                    "Monthly rollover must be RUNNING before failure: $ownerId/$yearMonth"
                }

                val updated =
                    current.copy(
                        status = MonthlyRolloverState.Status.FAILED,
                        failedAt = failedAt,
                        error = error
                    )

                transaction.set(
                    rolloverReference,
                    toDocument(updated)
                )

                /*
                 * Owner remains discoverable for the normal
                 * recovery job.
                 */
                transaction.set(
                    retryReference,
                    mapOf(
                        "ownerId" to ownerId,
                        "yearMonth" to yearMonth,
                        "status" to MonthlyRolloverState.Status.FAILED.name,
                        "attempt" to updated.attempt,
                        "failedAt" to failedAt.let(::toTimestamp),
                        "error" to error
                    )
                )

                updated
            }
            .get()
    }

    fun findFailedOwners(
        yearMonth: String
    ): List<String> {

        validateYearMonth(yearMonth)

        val snapshot =
            firestore
                .collection("system")
                .document("monthly_rollover_retry")
                .collection(yearMonth)
                .whereEqualTo(
                    "status",
                    MonthlyRolloverState.Status.FAILED.name
                )
                .get()
                .get()

        return snapshot.documents.map { document ->
            document.getString("ownerId")
                ?: throw IllegalStateException(
                    "Monthly rollover retry entry is missing ownerId: ${document.id}"
                )
        }
    }

    private fun toDocument(
        state: MonthlyRolloverState
    ): Map<String, Any?> {

        return mapOf(
            "ownerId" to state.ownerId,
            "yearMonth" to state.yearMonth,
            "status" to state.status.name,
            "attempt" to state.attempt,
            "startedAt" to state.startedAt?.let(::toTimestamp),
            "completedAt" to state.completedAt?.let(::toTimestamp),
            "failedAt" to state.failedAt?.let(::toTimestamp),
            "error" to state.error
        )
    }

    private fun toState(
        snapshot: DocumentSnapshot
    ): MonthlyRolloverState {

        val ownerId =
            snapshot.getString("ownerId")
                ?: throw IllegalStateException(
                    "Monthly rollover state is missing ownerId"
                )

        val yearMonth =
            snapshot.getString("yearMonth")
                ?: throw IllegalStateException(
                    "Monthly rollover state is missing yearMonth"
                )

        val status =
            snapshot.getString("status")
                ?.let(MonthlyRolloverState.Status::valueOf)
                ?: throw IllegalStateException(
                    "Monthly rollover state is missing status"
                )

        return MonthlyRolloverState(
            ownerId = ownerId,
            yearMonth = yearMonth,
            status = status,
            attempt = snapshot.getLong("attempt") ?: 0,
            startedAt = snapshot.getTimestamp("startedAt")
                ?.toDate()
                ?.toInstant(),
            completedAt = snapshot.getTimestamp("completedAt")
                ?.toDate()
                ?.toInstant(),
            failedAt = snapshot.getTimestamp("failedAt")
                ?.toDate()
                ?.toInstant(),
            error = snapshot.getString("error")
        )
    }

    private fun toTimestamp(
        instant: Instant
    ): Timestamp =
        Timestamp.ofTimeSecondsAndNanos(
            instant.epochSecond,
            instant.nano
        )

    private fun validateOwnerId(
        ownerId: String
    ) {
        require(ownerId.isNotBlank()) {
            "ownerId must not be blank"
        }
    }

    private fun validateYearMonth(
        yearMonth: String
    ) {
        require(yearMonth.matches(Regex("\\d{4}-\\d{2}"))) {
            "yearMonth must be in yyyy-MM format"
        }
    }
}
