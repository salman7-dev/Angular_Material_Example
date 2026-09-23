package com.clientledger.core.service

interface CurrentOwnerProvider {
    fun getOwnerId(): String
}
