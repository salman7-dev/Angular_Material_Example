--app
import org.gradle.api.tasks.testing.Test

plugins {
    id("buildlogic.kotlin-application-conventions")
    kotlin("plugin.spring") version "2.0.0"
    id("org.springframework.boot") version "3.2.0"
    id("io.spring.dependency-management") version "1.1.4"
}

dependencies {
    implementation("org.apache.commons:commons-text")
    implementation(project(":utilities"))

    testImplementation("com.h2database:h2:2.2.224")

    testImplementation("org.junit.jupiter:junit-jupiter-api:5.10.0")
    testRuntimeOnly("org.junit.jupiter:junit-jupiter-engine:5.10.0")

    testImplementation("org.mockito.kotlin:mockito-kotlin:5.4.0")

    implementation("com.google.firebase:firebase-admin:9.2.0")

    implementation("org.springframework.boot:spring-boot-starter-web")

    implementation("org.springframework.boot:spring-boot-starter-security")

    implementation("com.fasterxml.jackson.module:jackson-module-kotlin")

    implementation("org.apache.pdfbox:pdfbox:3.0.5")

    // Add these two lines:
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.7.3")
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-reactor:1.7.3")

    implementation("org.reactivestreams:reactive-streams:1.0.4")

    testImplementation("org.springframework.boot:spring-boot-starter-test")
}

tasks.withType<Test> {
    useJUnitPlatform()

    environment(
        "FIREBASE_AUTH_EMULATOR_HOST",
        System.getenv("FIREBASE_AUTH_EMULATOR_HOST")
            ?: "127.0.0.1:9099"
    )
}

application {
    mainClass.set("com.clientledger.core.app.ClientLedgerApplicationKt")
}
