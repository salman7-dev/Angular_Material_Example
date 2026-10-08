public class Main {

    public static void main(String[] args) throws Exception {

        String excelPath =
                "D:\\TestData\\ClientLedger\\orders.xlsx";

        String firebaseToken =
                "YOUR_FIREBASE_TOKEN";

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
