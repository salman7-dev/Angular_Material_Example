package com.clientledger.core.auth

import com.clientledger.core.config.ClientLedgerProperties
import jakarta.servlet.FilterChain
import jakarta.servlet.http.HttpServletRequest
import jakarta.servlet.http.HttpServletResponse
import org.slf4j.LoggerFactory
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken
import org.springframework.security.core.authority.AuthorityUtils
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
        LoggerFactory.getLogger(FirebaseAuthenticationFilter::class.java)

    override fun doFilterInternal(
        request: HttpServletRequest,
        response: HttpServletResponse,
        filterChain: FilterChain
    ) {
        if (!properties.auth.enabled) {
            filterChain.doFilter(request, response)
            return
        }

        try {
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
                authorization.removePrefix("Bearer ").trim()

            if (token.isBlank()) {
                response.sendError(
                    HttpServletResponse.SC_UNAUTHORIZED,
                    "Firebase ID token must not be blank"
                )
                return
            }

            val ownerId =
                firebaseTokenService.verifyToken(token)

            currentOwnerContext.setOwnerId(ownerId)

            SecurityContextHolder
                .getContext()
                .authentication =
                UsernamePasswordAuthenticationToken(
                    ownerId,
                    null,
                    AuthorityUtils.NO_AUTHORITIES
                )

            filterChain.doFilter(request, response)

        } catch (ex: Exception) {

            logger.error(
                "Firebase authentication failed",
                ex
            )

            SecurityContextHolder.clearContext()

            response.sendError(
                HttpServletResponse.SC_UNAUTHORIZED,
                "Invalid Firebase ID token"
            )
        } finally {
            currentOwnerContext.clear()
        }
    }
}
