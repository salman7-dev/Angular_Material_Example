plugins {
	kotlin("jvm") version "2.0.20"
	kotlin("plugin.spring") version "2.0.20"
	id("org.springframework.boot") version "3.3.4"
	id("io.spring.dependency-management") version "1.1.6"
}

group = "com.clientledger"
version = "0.0.1-SNAPSHOT"

java {
	toolchain {
		languageVersion = JavaLanguageVersion.of(21)
	}
}

repositories {
	mavenCentral()
}

dependencies {
	// Web & Tomcat Server
	implementation("org.springframework.boot:spring-boot-starter-web")
	implementation("org.springframework.boot:spring-boot-starter-tomcat")
	implementation("org.springframework.boot:spring-boot-starter-security")
	implementation("org.springframework.boot:spring-boot-starter-validation")
	implementation("org.springframework.boot:spring-boot-starter-actuator")

	// Kotlin & Jackson
	implementation("org.jetbrains.kotlin:kotlin-reflect")
	implementation("com.fasterxml.jackson.module:jackson-module-kotlin")

	// Third Party Libraries
	implementation("com.google.firebase:firebase-admin:9.3.0") {
		exclude(group = "commons-logging", module = "commons-logging")
	}
	implementation("com.google.cloud:google-cloud-firestore:3.21.0")
	implementation("org.apache.commons:commons-text:1.11.0")
	implementation("org.apache.pdfbox:pdfbox:3.0.5") {
		exclude(group = "commons-logging", module = "commons-logging")
	}

	// Reactive & Coroutines
	implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core")
	implementation("org.jetbrains.kotlinx:kotlinx-coroutines-reactor")
	implementation("org.reactivestreams:reactive-streams")

	// Testing
	testImplementation("org.springframework.boot:spring-boot-starter-test")
	testImplementation("org.springframework.security:spring-security-test")
	testImplementation("org.jetbrains.kotlin:kotlin-test-junit5")
	testImplementation("com.h2database:h2")
	testImplementation("io.mockk:mockk:1.13.12")
	testImplementation("org.mockito.kotlin:mockito-kotlin:5.4.0")

	testRuntimeOnly("org.junit.platform:junit-platform-launcher")
}

kotlin {
	compilerOptions {
		freeCompilerArgs.addAll("-Xjsr305=strict")
	}
}
