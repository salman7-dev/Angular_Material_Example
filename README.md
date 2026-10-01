package com.clientledger.core.pagination

data class PageResult<T>(
    val content: List<T>,
    val hasNext: Boolean,
    val nextCursor: String?
)
