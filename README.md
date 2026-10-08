package com.clientledger.loadtest;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;

import java.io.BufferedReader;
import java.io.IOException;
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.nio.file.Files;
import java.nio.file.Path;
import java.time.Duration;
import java.util.ArrayList;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

public class OrderExcelApiLoader {

    private static final ObjectMapper OBJECT_MAPPER = new ObjectMapper();

    private static final String DEFAULT_BASE_URL =
            "http://localhost:8081";

    private static final String ORDERS_ENDPOINT =
            "/api/orders";

    public static void main(String[] args) throws Exception {

        if (args.length < 2) {
            System.out.println("""
                    
                    Usage:
                    
                    java OrderExcelApiLoader <csv-file> <firebase-token>
                    
                    Example:
                    
                    java OrderExcelApiLoader orders.csv "eyJhbGciOi..."
                    
                    Optional third argument:
                    
                    <base-url>
                    
                    Example:
                    
                    java OrderExcelApiLoader orders.csv "eyJ..." http://localhost:8081
                    """);

            return;
        }

        Path csvFile = Path.of(args[0]);
        String firebaseToken = args[1];

        String baseUrl = args.length >= 3
                ? args[2]
                : DEFAULT_BASE_URL;

        validateInput(csvFile, firebaseToken);

        System.out.println("==========================================");
        System.out.println(" Client Ledger API Order Loader");
        System.out.println("==========================================");
        System.out.println("CSV File  : " + csvFile.toAbsolutePath());
        System.out.println("Base URL  : " + baseUrl);
        System.out.println();

        List<OrderRow> rows = readCsv(csvFile);

        if (rows.isEmpty()) {
            System.out.println("No data found in CSV.");
            return;
        }

        Map<String, List<OrderRow>> orders =
                groupByOrder(rows);

        System.out.println("Total item rows : " + rows.size());
        System.out.println("Total orders    : " + orders.size());
        System.out.println();

        HttpClient httpClient = HttpClient.newBuilder()
                .connectTimeout(Duration.ofSeconds(10))
                .build();

        LoadTestResult result =
                callOrdersApi(
                        httpClient,
                        baseUrl,
                        firebaseToken,
                        orders
                );

        printSummary(result);
    }

    private static List<OrderRow> readCsv(Path csvFile)
            throws IOException {

        List<OrderRow> rows = new ArrayList<>();

        try (BufferedReader reader =
                     Files.newBufferedReader(csvFile)) {

            String header = reader.readLine();

            if (header == null) {
                return rows;
            }

            String line;

            int lineNumber = 1;

            while ((line = reader.readLine()) != null) {

                lineNumber++;

                if (line.isBlank()) {
                    continue;
                }

                String[] values = line.split(",", -1);

                if (values.length != 11) {
                    throw new IllegalArgumentException(
                            "Invalid CSV at line "
                                    + lineNumber
                                    + ". Expected 11 columns but found "
                                    + values.length
                    );
                }

                rows.add(
                        new OrderRow(
                                values[0].trim(),
                                values[1].trim(),
                                values[2].trim(),
                                values[3].trim(),
                                values[4].trim(),
                                Integer.parseInt(values[5].trim()),
                                values[6].trim(),
                                Long.parseLong(values[7].trim()),
                                Integer.parseInt(values[8].trim()),
                                Long.parseLong(values[9].trim()),
                                Long.parseLong(values[10].trim())
                        )
                );
            }
        }

        return rows;
    }

    private static Map<String, List<OrderRow>> groupByOrder(
            List<OrderRow> rows) {

        Map<String, List<OrderRow>> grouped =
                new LinkedHashMap<>();

        for (OrderRow row : rows) {

            grouped
                    .computeIfAbsent(
                            row.orderNo(),
                            key -> new ArrayList<>()
                    )
                    .add(row);
        }

        return grouped;
    }

    private static LoadTestResult callOrdersApi(
            HttpClient httpClient,
            String baseUrl,
            String firebaseToken,
            Map<String, List<OrderRow>> orders
    ) throws Exception {

        int success = 0;
        int failure = 0;

        List<Long> responseTimes = new ArrayList<>();

        for (Map.Entry<String, List<OrderRow>> entry
                : orders.entrySet()) {

            String orderNo = entry.getKey();

            List<OrderRow> rows = entry.getValue();

            OrderRequest request =
                    createOrderRequest(rows);

            String json =
                    OBJECT_MAPPER.writeValueAsString(request);

            System.out.println("------------------------------------------");
            System.out.println("Order: " + orderNo);
            System.out.println("Items: " + rows.size());

            long start = System.nanoTime();

            HttpRequest httpRequest =
                    HttpRequest.newBuilder()
                            .uri(
                                    URI.create(
                                            baseUrl + ORDERS_ENDPOINT
                                    )
                            )
                            .timeout(Duration.ofSeconds(60))
                            .header(
                                    "Authorization",
                                    "Bearer " + firebaseToken
                            )
                            .header(
                                    "Content-Type",
                                    "application/json"
                            )
                            .POST(
                                    HttpRequest.BodyPublishers.ofString(
                                            json
                                    )
                            )
                            .build();

            HttpResponse<String> response =
                    httpClient.send(
                            httpRequest,
                            HttpResponse.BodyHandlers.ofString()
                    );

            long responseTimeMs =
                    (System.nanoTime() - start) / 1_000_000;

            responseTimes.add(responseTimeMs);

            System.out.println(
                    "HTTP Status   : "
                            + response.statusCode()
            );

            System.out.println(
                    "Response Time : "
                            + responseTimeMs
                            + " ms"
            );

            if (response.statusCode() >= 200
                    && response.statusCode() < 300) {

                success++;

                printApiTotals(response.body());

                System.out.println("Result        : SUCCESS");

            } else {

                failure++;

                System.out.println("Result        : FAILED");

                System.out.println(
                        "Response Body : "
                                + response.body()
                );

                System.out.println(
                        "Request Body  : "
                                + json
                );
            }
        }

        return new LoadTestResult(
                orders.size(),
                success,
                failure,
                responseTimes
        );
    }

    private static OrderRequest createOrderRequest(
            List<OrderRow> rows) {

        if (rows.isEmpty()) {
            throw new IllegalArgumentException(
                    "Order must contain at least one item"
            );
        }

        OrderRow first = rows.getFirst();

        List<OrderItemRequest> items =
                new ArrayList<>();

        for (OrderRow row : rows) {

            items.add(
                    new OrderItemRequest(
                            row.itemName(),
                            row.quantity(),
                            row.unit(),
                            row.amount(),
                            row.gstRate(),
                            row.gstAmount(),
                            row.expense()
                    )
            );
        }

        return new OrderRequest(
                first.clientId(),
                first.orderDate(),
                first.deliveryDate(),
                items
        );
    }

    private static void printApiTotals(
            String responseBody) {

        try {

            JsonNode json =
                    OBJECT_MAPPER.readTree(responseBody);

            System.out.println(
                    "API Order ID  : "
                            + json.path("id").asText()
            );

            System.out.println(
                    "Invoice Amount: "
                            + json.path(
                                    "totalInvoiceAmount"
                            ).asLong()
            );

            System.out.println(
                    "Amount + GST  : "
                            + json.path(
                                    "totalInvoiceAmountWithGst"
                            ).asLong()
            );

            System.out.println(
                    "GST Amount    : "
                            + json.path(
                                    "totalGstAmount"
                            ).asLong()
            );

            System.out.println(
                    "Expense       : "
                            + json.path(
                                    "totalExpense"
                            ).asLong()
            );

            System.out.println(
                    "Profit        : "
                            + json.path(
                                    "profitAmount"
                            ).asLong()
            );

        } catch (Exception ignored) {

            System.out.println(
                    "API Response   : "
                            + responseBody
            );
        }
    }

    private static void printSummary(
            LoadTestResult result) {

        System.out.println();
        System.out.println("==========================================");
        System.out.println(" FINAL SUMMARY");
        System.out.println("==========================================");

        System.out.println(
                "Total Orders : "
                        + result.totalOrders()
        );

        System.out.println(
                "Success      : "
                        + result.success()
        );

        System.out.println(
                "Failed       : "
                        + result.failure()
        );

        if (!result.responseTimes().isEmpty()) {

            long min =
                    result.responseTimes()
                            .stream()
                            .mapToLong(Long::longValue)
                            .min()
                            .orElse(0);

            long max =
                    result.responseTimes()
                            .stream()
                            .mapToLong(Long::longValue)
                            .max()
                            .orElse(0);

            double average =
                    result.responseTimes()
                            .stream()
                            .mapToLong(Long::longValue)
                            .average()
                            .orElse(0);

            System.out.println(
                    "Min Response : "
                            + min
                            + " ms"
            );

            System.out.println(
                    "Avg Response : "
                            + String.format(
                                    "%.2f",
                                    average
                            )
                            + " ms"
            );

            System.out.println(
                    "Max Response : "
                            + max
                            + " ms"
            );
        }

        System.out.println("==========================================");
    }

    private static void validateInput(
            Path csvFile,
            String firebaseToken) {

        if (!Files.exists(csvFile)) {
            throw new IllegalArgumentException(
                    "CSV file does not exist: "
                            + csvFile
            );
        }

        if (firebaseToken == null
                || firebaseToken.isBlank()) {

            throw new IllegalArgumentException(
                    "Firebase token must not be blank"
            );
        }
    }

    record OrderRow(
            String orderNo,
            String clientId,
            String orderDate,
            String deliveryDate,
            String itemName,
            int quantity,
            String unit,
            long amount,
            int gstRate,
            long gstAmount,
            long expense
    ) {
    }

    record OrderRequest(
            String clientId,
            String orderDate,
            String deliveryDate,
            List<OrderItemRequest> items
    ) {
    }

    record OrderItemRequest(
            String name,
            int quantity,
            String unit,
            long amount,
            int gstRate,
            long gstAmount,
            long expense
    ) {
    }

    record LoadTestResult(
            int totalOrders,
            int success,
            int failure,
            List<Long> responseTimes
    ) {
    }
}
