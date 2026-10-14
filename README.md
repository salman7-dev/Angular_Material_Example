package com.clientledger.core.config

import com.fasterxml.jackson.core.JsonParser
import com.fasterxml.jackson.databind.DeserializationContext
import com.fasterxml.jackson.databind.JsonDeserializer
import java.math.BigDecimal
import java.math.RoundingMode

class MoneyDeserializer : JsonDeserializer<BigDecimal>() {

    override fun deserialize(
        parser: JsonParser,
        context: DeserializationContext
    ): BigDecimal {
        return BigDecimal(parser.text)
            .setScale(2, RoundingMode.HALF_UP)
    }
}
