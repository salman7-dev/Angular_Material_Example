package com.clientledger.core.auth

import com.clientledger.core.auth.dto.AuthResponse
import com.clientledger.core.auth.dto.LoginRequest
import com.clientledger.core.auth.dto.RefreshTokenRequest
import com.clientledger.core.auth.dto.SignupRequest
import jakarta.validation.Valid
import org.springframework.http.HttpStatus
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/api/auth")
class AuthController(
    private val authService: AuthService
) {

    @PostMapping("/signup")
    @ResponseStatus(HttpStatus.CREATED)
    fun signup(
        @Valid @RequestBody request: SignupRequest
    ): AuthResponse {
        return authService.signup(request)
    }

    @PostMapping("/login")
    fun login(
        @Valid @RequestBody request: LoginRequest
    ): AuthResponse {
        return authService.login(request)
    }

    @PostMapping("/refresh")
    fun refresh(
        @Valid @RequestBody request: RefreshTokenRequest
    ): AuthResponse {
        return authService.refresh(request)
    }

    @GetMapping("/status")
    fun status(
       ): String {
        return "Server is running.."
    }
}
