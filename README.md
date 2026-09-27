{
    "kind": "identitytoolkit#SignupNewUserResponse",
    "localId": "wfcOBhPqX4xxI3utPsHtkXCp1zhJ",
    "email": "owner1@example.com",
    "idToken": "eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0.eyJlbWFpbCI6Im93bmVyMUBleGFtcGxlLmNvbSIsImVtYWlsX3ZlcmlmaWVkIjpmYWxzZSwiYXV0aF90aW1lIjoxNzkwNDk4NDk2LCJ1c2VyX2lkIjoid2ZjT0JoUHFYNHh4STN1dFBzSHRrWENwMXpoSiIsImZpcmViYXNlIjp7ImlkZW50aXRpZXMiOnsiZW1haWwiOlsib3duZXIxQGV4YW1wbGUuY29tIl19LCJzaWduX2luX3Byb3ZpZGVyIjoicGFzc3dvcmQifSwiaWF0IjoxNzkwNDk4NDk2LCJleHAiOjE3OTA1MDIwOTYsImF1ZCI6ImNsaWVudC1sZWRnZXItZGFzaGJvYXJkIiwiaXNzIjoiaHR0cHM6Ly9zZWN1cmV0b2tlbi5nb29nbGUuY29tL2NsaWVudC1sZWRnZXItZGFzaGJvYXJkIiwic3ViIjoid2ZjT0JoUHFYNHh4STN1dFBzSHRrWENwMXpoSiJ9.",
    "refreshToken": "eyJfQXV0aEVtdWxhdG9yUmVmcmVzaFRva2VuIjoiRE8gTk9UIE1PRElGWSIsImxvY2FsSWQiOiJ3ZmNPQmhQcVg0eHhJM3V0UHNIdGtYQ3AxemhKIiwicHJvdmlkZXIiOiJwYXNzd29yZCIsImV4dHJhQ2xhaW1zIjp7fSwicHJvamVjdElkIjoiY2xpZW50LWxlZGdlci1kYXNoYm9hcmQifQ==",
    "expiresIn": "3600"
}


package com.clientledger.core.auth

import com.clientledger.core.config.ClientLedgerProperties
import jakarta.servlet.FilterChain
import jakarta.servlet.http.HttpServletRequest
import jakarta.servlet.http.HttpServletResponse
import org.slf4j.LoggerFactory
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken
import org.springframework.security.core.authority.SimpleGrantedAuthority
import org.springframework.security.core.context.SecurityContextHolder
import org.springframework.stereotype.Component
import org.springframework.web.filter.OncePerRequestFilter

@Component
class FirebaseAuthenticationFilter(
    private val firebaseTokenService: FirebaseTokenService,
    private val currentOwnerContext: CurrentOwnerContext,
    private val properties: ClientLedgerProperties
) : OncePerRequestFilter() {

    private val logger =
        LoggerFactory.getLogger(
            FirebaseAuthenticationFilter::class.java
        )

    override fun doFilterInternal(
        request: HttpServletRequest,
        response: HttpServletResponse,
        filterChain: FilterChain
    ) {
        if (!properties.auth.enabled) {
            filterChain.doFilter(request, response)
            return
        }

        val authorization =
            request.getHeader("Authorization")

        if (authorization.isNullOrBlank()) {
            response.sendError(
                HttpServletResponse.SC_UNAUTHORIZED,
                "Authorization header is required"
            )
            return
        }

        if (!authorization.startsWith("Bearer ")) {
            response.sendError(
                HttpServletResponse.SC_UNAUTHORIZED,
                "Invalid Authorization header"
            )
            return
        }

        val token =
            authorization
                .removePrefix("Bearer ")
                .trim()

        if (token.isBlank()) {
            response.sendError(
                HttpServletResponse.SC_UNAUTHORIZED,
                "Firebase ID token must not be blank"
            )
            return
        }

        val authenticatedUser =
            try {
                firebaseTokenService.verifyToken(token)
            } catch (ex: Exception) {

                logger.warn("Firebase authentication failed: {}", ex)

                response.sendError(
                    HttpServletResponse.SC_UNAUTHORIZED,
                    "Invalid Firebase ID token"
                )

                return
            }

        currentOwnerContext.setOwnerId(
            authenticatedUser.uid
        )

        val authority =
            SimpleGrantedAuthority(
                "ROLE_${authenticatedUser.role.name}"
            )

        SecurityContextHolder
            .getContext()
            .authentication =
            UsernamePasswordAuthenticationToken(
                authenticatedUser.uid,
                null,
                listOf(authority)
            )

        try {
            filterChain.doFilter(
                request,
                response
            )
        } finally {
            SecurityContextHolder.clearContext()
            currentOwnerContext.clear()
        }
    }
}
