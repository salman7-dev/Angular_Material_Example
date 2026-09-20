we can use thie utils to genearte unique id package com.clientledger.core.utils

import java.security.SecureRandom

object IdGenerator {

    private const val ALPHABET = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ"
    private val random = SecureRandom()

    private fun generateId(prefix: String): String {
        val timestamp = System.currentTimeMillis().toString(36).uppercase()
        val randomTail = ALPHABET[random.nextInt(ALPHABET.length)]
        return "$prefix-$timestamp$randomTail"
    }

    fun generateClientId(): String = generateId("CLI")
    fun generateOrderId(): String = generateId("ORD")
    fun generatePaymentId(): String = generateId("PAY")
    fun generateInvoiceId(): String = generateId("INV")
    fun generateBusinessExpense(): String = generateId("BUS-EXP")
    fun generateOwnerId():String = generateId("OWN")
}
