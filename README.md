package com.clientledger.core.config

import com.clientledger.core.auth.FirebaseAuthenticationFilter
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.security.config.annotation.web.builders.HttpSecurity
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity
import org.springframework.security.config.http.SessionCreationPolicy
import org.springframework.security.web.SecurityFilterChain
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter

@Configuration
@EnableWebSecurity
class SecurityConfig(
    private val firebaseAuthenticationFilter: FirebaseAuthenticationFilter,
    private val properties: ClientLedgerProperties
) {

    @Bean
    fun securityFilterChain(
        http: HttpSecurity
    ): SecurityFilterChain {

        http
            .csrf { it.disable() }
            .sessionManagement {
                it.sessionCreationPolicy(
                    SessionCreationPolicy.STATELESS
                )
            }
            .authorizeHttpRequests { auth ->

                if (!properties.auth.enabled) {

                    auth.anyRequest().permitAll()

                } else {

                    auth
                        .requestMatchers(
                            "/actuator/health",
                            "/api/auth/**"
                        )
                        .permitAll()

                        // GENERAL users can access the normal APIs.
                        .requestMatchers(
                            "/api/clients/**",
                            "/api/orders/**",
                            "/api/payments/**",
                            "/api/invoices/**",
                            "/api/summary/**"
                        )
                        .hasAnyRole(
                            "GENERAL",
                            "PRO"
                        )

                        // PRO-only APIs will be added here.
                        // Example:
                        .requestMatchers("/api/pro/**")
                        .hasRole("PRO")

                        .anyRequest()
                        .authenticated()
                }
            }

        if (properties.auth.enabled) {
            http.addFilterBefore(
                firebaseAuthenticationFilter,
                UsernamePasswordAuthenticationFilter::class.java
            )
        }

        return http.build()
    }
}
