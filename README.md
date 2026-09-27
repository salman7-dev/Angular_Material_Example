package com.clientledger.core.service.owner

import com.clientledger.core.domain.owner.Owner
import com.clientledger.core.repository.owner.OwnerRepository
import com.clientledger.core.utils.IdGenerator
import org.springframework.stereotype.Service
import java.time.Instant

@Service
class OwnerService(
    private val ownerRepository: OwnerRepository
) {

    fun createOwner(
        name: String,
        businessName: String,
        phone: String,
        email: String,
        gstNumber: String,
        address: com.clientledger.core.domain.Address
    ): Owner {

        require(name.isNotBlank()) {
            "name must not be blank"
        }

        require(businessName.isNotBlank()) {
            "businessName must not be blank"
        }

        val now = Instant.now()

        val owner = Owner(
            ownerId = IdGenerator.generateClientId(),
            name = name,
            businessName = businessName,
            phone = phone,
            email = email,
            gstNumber = gstNumber,
            address = address,
            createdAt = now,
            updatedAt = now
        )

        return ownerRepository.create(owner)
    }

    fun getOwner(
        ownerId: String
    ): Owner? {

        require(ownerId.isNotBlank()) {
            "ownerId must not be blank"
        }

        return ownerRepository.find(ownerId)
    }
}
