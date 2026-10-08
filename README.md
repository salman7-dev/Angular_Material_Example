package com.clientledger.loader;

public class Main {

    public static void main(String[] args) throws Exception {

        String excelPath =
                "D:\\TestData\\ClientLedger\\orders.xlsx";

        String firebaseToken =
                "YOUR_FIREBASE_ID_TOKEN";

        String baseUrl =
                "http://localhost:8081";

        OrderExcelApiLoader loader =
                new OrderExcelApiLoader(
                        excelPath,
                        firebaseToken,
                        baseUrl
                );

        loader.run();
    }
}



package com.clientledger.loader;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.apache.poi.ss.usermodel.*;
import org.apache.poi.xssf.usermodel.XSSFWorkbook;

import java.io.FileInputStream;
import java.io.IOException;
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.time.Duration;
import java.util.ArrayList;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

public class OrderExcelApiLoader {

    private final String excelPath;
    private final String firebaseToken;
    private final String baseUrl;

    private final ObjectMapper objectMapper;
    private final HttpClient httpClient;

    public OrderExcelApiLoader(
            String excelPath,
            String firebaseToken,
            String baseUrl) {

        this.excelPath = excelPath;
        this.firebaseToken = firebaseToken;
        this.baseUrl = baseUrl;

        this.objectMapper = new ObjectMapper();

        this.httpClient = HttpClient.newBuilder()
                .connectTimeout(Duration.ofSeconds(10))
                .build();
    }

    public void run() throws Exception {

        System.out.println("======================================");
        System.out.println(" Client Ledger Excel API Loader");
        System.out.println("======================================");

        System.out.println("Excel : " + excelPath);
        System.out.println("API   : " + baseUrl);
        System.out.println();

        List<OrderRow> rows = readExcel();

        System.out.println("Excel item rows : " + rows.size());

        Map<String, List<OrderRow>> orders =
                groupOrders(rows);

        System.out.println(
                "Excel orders    : " + orders.size()
        );

        System.out.println();

        loadOrders(orders);
    }

    private List<OrderRow> readExcel()
            throws IOException {

        List<OrderRow> rows = new ArrayList<>();

        try (
                FileInputStream inputStream =
                        new FileInputStream(excelPath);

                Workbook workbook =
                        new XSSFWorkbook(inputStream)
        ) {

            Sheet sheet = workbook.getSheetAt(0);

            boolean header = true;

            for (Row row : sheet) {

                if (header) {
                    header = false;
                    continue;
                }

                if (isEmptyRow(row)) {
                    continue;
                }

                OrderRow orderRow =
                        readOrderRow(row);

                rows.add(orderRow);
            }
        }

        return rows;
    }

    private OrderRow readOrderRow(Row row) {

        return new OrderRow(
                getString(row, 0),
                getString(row, 1),
                getString(row, 2),
                getString(row, 3),
                getString(row, 4),
                getInt(row, 5),
                getString(row, 6),
                getLong(row, 7),
                getInt(row, 8),
                getLong(row, 9),
                getLong(row, 10)
        );
    }

    private Map<String, List<OrderRow>> groupOrders(
            List<OrderRow> rows) {

        Map<String, List<OrderRow>> orders =
                new LinkedHashMap<>();

        for (OrderRow row : rows) {

            orders
                    .computeIfAbsent(
                            row.orderNo(),
                            key -> new ArrayList<>()
                    )
                    .add(row);
        }

        return orders;
    }

    private void loadOrders(
            Map<String, List<OrderRow>> orders)
            throws Exception {

        int success = 0;
        int failed = 0;

        for (Map.Entry<String, List<OrderRow>> entry
                : orders.entrySet()) {

            String orderNo = entry.getKey();

            List<OrderRow> rows = entry.getValue();

            System.out.println("--------------------------------------");
            System.out.println("Order No : " + orderNo);
            System.out.println("Items    : " + rows.size());

            validateOrderRows(orderNo, rows);

            OrderRequest request =
                    createRequest(rows);

            String requestJson =
                    objectMapper.writeValueAsString(request);

            System.out.println("Request:");
            System.out.println(requestJson);

            long startTime =
                    System.nanoTime();

            HttpRequest httpRequest =
                    HttpRequest.newBuilder()
                            .uri(
                                    URI.create(
                                            baseUrl + "/api/orders"
                                    )
                            )
                            .timeout(
                                    Duration.ofSeconds(60)
                            )
                            .header(
                                    "Authorization",
                                    "Bearer " + firebaseToken
                            )
                            .header(
                                    "Content-Type",
                                    "application/json"
                            )
                            .POST(
                                    HttpRequest.BodyPublishers
                                            .ofString(requestJson)
                            )
                            .build();

            HttpResponse<String> response =
                    httpClient.send(
                            httpRequest,
                            HttpResponse.BodyHandlers.ofString()
                    );

            long responseTime =
                    (System.nanoTime() - startTime)
                            / 1_000_000;

            System.out.println(
                    "HTTP Status : "
                            + response.statusCode()
            );

            System.out.println(
                    "Response Time : "
                            + responseTime
                            + " ms"
            );

            if (response.statusCode() >= 200
                    && response.statusCode() < 300) {

                success++;

                System.out.println("Result : SUCCESS");

                printResponse(response.body());

            } else {

                failed++;

                System.out.println("Result : FAILED");

                System.out.println(
                        "Response : "
                                + response.body()
                );
            }
        }

        System.out.println();
        System.out.println("======================================");
        System.out.println(" FINAL RESULT");
        System.out.println("======================================");

        System.out.println(
                "Total Orders : " + orders.size()
        );

        System.out.println(
                "Success      : " + success
        );

        System.out.println(
                "Failed       : " + failed
        );

        System.out.println("======================================");
    }

    private OrderRequest createRequest(
            List<OrderRow> rows) {

        OrderRow first = rows.get(0);

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

    private void validateOrderRows(
            String orderNo,
            List<OrderRow> rows) {

        OrderRow first = rows.get(0);

        for (OrderRow row : rows) {

            if (!first.clientId()
                    .equals(row.clientId())) {

                throw new IllegalArgumentException(
                        "Order "
                                + orderNo
                                + " has different clientId values"
                );
            }

            if (!first.orderDate()
                    .equals(row.orderDate())) {

                throw new IllegalArgumentException(
                        "Order "
                                + orderNo
                                + " has different orderDate values"
                );
            }

            if (!first.deliveryDate()
                    .equals(row.deliveryDate())) {

                throw new IllegalArgumentException(
                        "Order "
                                + orderNo
                                + " has different deliveryDate values"
                );
            }
        }
    }

    private void printResponse(
            String responseBody) {

        try {

            JsonNode json =
                    objectMapper.readTree(responseBody);

            System.out.println(
                    "API Order ID : "
                            + json.path("id").asText()
            );

            System.out.println(
                    "Invoice Amount : "
                            + json.path(
                                    "totalInvoiceAmount"
                            ).asLong()
            );

            System.out.println(
                    "Amount + GST : "
                            + json.path(
                                    "totalInvoiceAmountWithGst"
                            ).asLong()
            );

            System.out.println(
                    "GST Amount : "
                            + json.path(
                                    "totalGstAmount"
                            ).asLong()
            );

            System.out.println(
                    "Expense : "
                            + json.path(
                                    "totalExpense"
                            ).asLong()
            );

            System.out.println(
                    "Profit : "
                            + json.path(
                                    "profitAmount"
                            ).asLong()
            );

        } catch (Exception e) {

            System.out.println(
                    "Response : "
                            + responseBody
            );
        }
    }

    private String getString(
            Row row,
            int column) {

        Cell cell = row.getCell(column);

        if (cell == null) {
            return "";
        }

        DataFormatter formatter =
                new DataFormatter();

        return formatter.formatCellValue(cell)
                .trim();
    }

    private int getInt(
            Row row,
            int column) {

        String value =
                getString(row, column);

        return Integer.parseInt(value);
    }

    private long getLong(
            Row row,
            int column) {

        String value =
                getString(row, column);

        return Long.parseLong(value);
    }

    private boolean isEmptyRow(Row row) {

        for (int i = 0; i < 11; i++) {

            if (!getString(row, i).isBlank()) {
                return false;
            }
        }

        return true;
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
}
