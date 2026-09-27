package com.clientledger.core.auth

import com.google.firebase.auth.FirebaseAuth
import org.springframework.stereotype.Service

@Service
class FirebaseTokenService(
    private val firebaseAuth: FirebaseAuth
) {

    fun verifyToken(idToken: String): AuthenticatedUser {

        require(idToken.isNotBlank()) {
            "Firebase ID token must not be blank"
        }

        val decodedToken =
            firebaseAuth.verifyIdToken(idToken)

        val role =
            decodedToken
                .claims["role"]
                ?.toString()
                ?.let { value ->
                    runCatching {
                        UserRole.valueOf(value)
                    }.getOrNull()
                }
                ?: UserRole.GENERAL

        return AuthenticatedUser(
            uid = decodedToken.uid,
            role = role
        )
    }
}
