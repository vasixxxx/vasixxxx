================================================================================
ЗАДАНИЕ: PATCH /v3/employees/{employee-id} — Обновить работника
================================================================================

Изменены 4 файла:
  1. src/main/kotlin/ru/yarsu/auth/AuthService.kt
  2. src/main/kotlin/ru/yarsu/storage/DataStorage.kt
  3. src/main/kotlin/ru/yarsu/routes/EmployeeRoutes.kt
  4. src/main/kotlin/ru/yarsu/Main.kt

================================================================================
ФАЙЛ 1: src/main/kotlin/ru/yarsu/auth/AuthService.kt
================================================================================

Добавлена функция authenticateUserManager — аутентификация только для роли UserManager.
Существующий authenticate() блокирует UserManager; новый метод, напротив, принимает
только UserManager и отклоняет всех остальных.

--- ДОБАВИТЬ после закрывающей скобки функции authenticate() ---

    fun authenticateUserManager(token: String?): Employee? {
        if (token == null) return null
        return try {
            val decoded = verifier.verify(token)
            val sub = decoded.subject ?: return null
            val id =
                try {
                    UUID.fromString(sub)
                } catch (_: IllegalArgumentException) {
                    return null
                }
            val employee = storage.getEmployeeById(id) ?: return null
            if (employee.role != Role.UserManager) return null
            employee
        } catch (_: JWTVerificationException) {
            null
        }
    }

================================================================================
ФАЙЛ 2: src/main/kotlin/ru/yarsu/storage/DataStorage.kt
================================================================================

Добавлена функция updateEmployee — обновляет запись работника в списке по id.

--- ДОБАВИТЬ после функции getEmployeeById() ---

    fun updateEmployee(employee: Employee): Boolean {
        val idx = employees.indexOfFirst { it.id == employee.id }
        if (idx < 0) return false
        employees[idx] = employee
        return true
    }

================================================================================
ФАЙЛ 3: src/main/kotlin/ru/yarsu/routes/EmployeeRoutes.kt
================================================================================

Файл переписан полностью. Добавлена зависимость AuthService и метод updateEmployee.
Метод getEmployees остался без изменений по логике, только добавлен параметр auth
в конструктор.

--- ПОЛНОЕ СОДЕРЖИМОЕ ФАЙЛА ---

package ru.yarsu.routes

import com.fasterxml.jackson.module.kotlin.readValue
import org.http4k.core.Request
import org.http4k.core.Response
import org.http4k.core.Status
import org.http4k.routing.path
import ru.yarsu.auth.AuthService
import ru.yarsu.storage.DataStorage
import java.util.UUID

class EmployeeRoutes(
    private val storage: DataStorage,
    private val auth: AuthService,
) {
    fun getEmployees(
        @Suppress("UNUSED_PARAMETER") request: Request,
    ) = jsonResponse(
        Status.OK,
        storage.getEmployees().map {
            mapOf(
                "Id" to it.id,
                "Name" to it.name,
                "Position" to it.position,
                "RegistrationDateTime" to it.registrationDateTime.toString(),
                "Email" to it.email,
            )
        },
    )

    fun updateEmployee(request: Request): Response {
        auth.authenticateUserManager(auth.extractToken(request.header("Authorization")))
            ?: return unauthorizedResponse()

        val idStr = request.path("employee-id")
        val id: UUID =
            try {
                UUID.fromString(idStr)
            } catch (_: IllegalArgumentException) {
                return jsonResponse(
                    Status.BAD_REQUEST,
                    mapOf("Value" to idStr, "Error" to "Некорректный идентификатор работника"),
                )
            }

        val bodyStr = request.bodyString()
        val body: Map<String, Any?>
        try {
            @Suppress("UNCHECKED_CAST")
            body = jsonMapper.readValue<Map<String, Any?>>(bodyStr)
        } catch (e: Exception) {
            return jsonResponse(
                Status.BAD_REQUEST,
                mapOf("Value" to bodyStr, "Error" to (e.message ?: "Некорректный JSON")),
            )
        }

        val nameRaw = body["Name"]
        if (nameRaw == null) {
            return jsonResponse(
                Status.BAD_REQUEST,
                mapOf("Name" to mapOf("Error" to "Поле Name обязательно")),
            )
        }
        if (nameRaw !is String) {
            return jsonResponse(
                Status.BAD_REQUEST,
                mapOf("Name" to mapOf("Error" to "Поле Name должно быть строкой")),
            )
        }
        val name = nameRaw.trim()
        if (name.isBlank()) {
            return jsonResponse(
                Status.BAD_REQUEST,
                mapOf("Name" to mapOf("Error" to "Поле Name не может быть пустым")),
            )
        }

        val existing = storage.getEmployeeById(id)
            ?: return jsonResponse(
                Status.NOT_FOUND,
                mapOf("EmployeeId" to id, "Error" to "Работник не найден"),
            )

        storage.updateEmployee(existing.copy(name = name))
        return Response(Status.NO_CONTENT)
    }
}

================================================================================
ФАЙЛ 4: src/main/kotlin/ru/yarsu/Main.kt
================================================================================

Изменение 1 — конструктор EmployeeRoutes теперь принимает authService:

package ru.yarsu

import com.github.ajalt.clikt.core.CliktCommand
import com.github.ajalt.clikt.parameters.options.option
import com.github.ajalt.clikt.parameters.options.required
import com.github.ajalt.clikt.parameters.types.int
import org.http4k.core.Method
import org.http4k.core.Response
import org.http4k.core.Status
import org.http4k.routing.bind
import org.http4k.routing.routes
import org.http4k.server.Netty
import org.http4k.server.asServer
import ru.yarsu.auth.AuthService
import ru.yarsu.routes.DumpTruckRoutes
import ru.yarsu.routes.EmployeeRoutes
import ru.yarsu.routes.ShipmentRoutes
import ru.yarsu.storage.DataStorage
import java.io.File

class App : CliktCommand() {
    private val swgFile by option("--swg-file", help = "Файл с данными отгрузок ПГС").required()
    private val dumpTrucksFile by option("--dump-trucks-file", help = "Файл с данными самосвалов").required()
    private val employeesFile by option("--employees-file", help = "Файл с данными работников").required()
    private val port by option("--port", help = "Порт веб-сервера").int().required()
    private val secret by option("--secret", help = "Секретный ключ для JWT HMAC512").required()

    override fun run() {
        validateFiles()

        val storage = DataStorage(swgFile, dumpTrucksFile, employeesFile)
        try {
            storage.load()
        } catch (e: Exception) {
            System.err.println("Error: не удалось загрузить данные: ${e.message}")
            System.exit(1)
        }

        val authService = AuthService(secret, storage)
        val shipmentRoutes = ShipmentRoutes(storage, authService)
        val dumpTruckRoutes = DumpTruckRoutes(storage, authService)
        val employeeRoutes = EmployeeRoutes(storage, authService)

        val app =
            routes(
                "/ping" bind Method.GET to { Response(Status.OK).body("pong") },
                "/v3/shipments/by-type" bind Method.GET to shipmentRoutes::getShipmentsByType,
                "/v3/shipments/by-period" bind Method.GET to shipmentRoutes::getShipmentsByPeriod,
                "/v3/shipments/report" bind Method.GET to shipmentRoutes::getReport,
                "/v3/shipments/{shipment-id}" bind Method.GET to shipmentRoutes::getShipment,
                "/v3/shipments/{shipment-id}" bind Method.PUT to shipmentRoutes::updateShipment,
                "/v3/shipments" bind Method.GET to shipmentRoutes::getShipments,
                "/v3/shipments" bind Method.POST to shipmentRoutes::createShipment,
                "/v3/register-invoice" bind Method.POST to shipmentRoutes::registerInvoice,
                "/v3/dump-trucks/{dump-truck-id}" bind Method.GET to dumpTruckRoutes::getDumpTruck,
                "/v3/dump-trucks/{dump-truck-id}" bind Method.DELETE to dumpTruckRoutes::deleteDumpTruck,
                "/v3/employees/{employee-id}" bind Method.PATCH to employeeRoutes::patchEmployee,
                "/v3/employees" bind Method.GET to employeeRoutes::getEmployees,
            )

        val server = app.asServer(Netty(port)).start()
        println("Сервер запущен на порту $port")

        Runtime.getRuntime().addShutdownHook(
            Thread {
                println("Завершение работы, сохранение данных...")
                try {
                    storage.save()
                    println("Данные сохранены")
                } catch (e: Exception) {
                    System.err.println("Ошибка при сохранении: ${e.message}")
                }
                server.stop()
            },
        )

        server.block()
    }

    private fun validateFiles() {
        val files =
            mapOf(
                "--swg-file" to swgFile,
                "--dump-trucks-file" to dumpTrucksFile,
                "--employees-file" to employeesFile,
            )
        for ((_, path) in files) {
            if (!File(path).exists()) {
                System.err.println("Error: file not found «$path»")
                System.exit(1)
            }
        }
    }
}

fun main(args: Array<String>) {
    App().main(args)
}


--- БЫЛО ---
        val employeeRoutes = EmployeeRoutes(storage)

--- СТАЛО ---
        val employeeRoutes = EmployeeRoutes(storage, authService)


Изменение 2 — добавлен маршрут PATCH /v3/employees/{employee-id}:

--- БЫЛО ---
                "/v3/employees" bind Method.GET to employeeRoutes::getEmployees,

--- СТАЛО ---
                "/v3/employees" bind Method.GET to employeeRoutes::getEmployees,
                "/v3/employees/{employee-id}" bind Method.PATCH to employeeRoutes::updateEmployee,



================================================================================
