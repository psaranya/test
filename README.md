@Service
public class ValidationService {

    @Autowired
    private EntityServiceClient entityServiceClient; // Client to call the entity service

    public boolean validateOpenCashStock(CashStocksActivateContextRequest request) {
        // 1. Validate CashStockId exists
        CashStock cashStock = entityServiceClient.getCashStock(Integer.parseInt(request.getCashStockId()));
        if (cashStock == null) {
            throw new IllegalArgumentException("CashStockId does not exist");
        }

        // 2. Validate CashStockStatusName is "CLOSE"
        if (!"CLOSE".equals(cashStock.getCashStockStatusName())) {
            throw new IllegalArgumentException("CashStockStatusName must be 'CLOSE'");
        }

        // 3. Validate CashStockSession exists
        CashStockSession cashStockSession = entityServiceClient.getCashStockSession(
            Integer.parseInt(cashStock.getCashStockLastSessionId()));
        if (cashStockSession == null) {
            throw new IllegalArgumentException("CashStockSession does not exist");
        }

        // 4. Fetch CashUnitStocks from the last session
        List<CashUnitStock> lastSessionCashUnitStocks = entityServiceClient.getCashUnitStocksBySession(
            Integer.parseInt(cashStock.getCashStockLastSessionId()));

        // 5. Validate that the provided CashUnitStocks match the last session
        Set<Integer> lastSessionCashUnitIds = lastSessionCashUnitStocks.stream()
            .map(CashUnitStock::getCashUnitId)
            .collect(Collectors.toSet());

        for (CashUnitStockContext cus : request.getCashUnitStockContextList()) {
            int cashUnitId = Integer.parseInt(cus.getCashUnitId());

            if (!lastSessionCashUnitIds.contains(cashUnitId)) {
                throw new IllegalArgumentException("CashUnitId " + cashUnitId + " is not registered in the last session.");
            }
        }

        // 6. Compute total cash value from provided CashUnitStocks
        int totalCashValue = request.getCashUnitStockContextList().stream()
            .mapToInt(cus -> {
                CashUnit cashUnit = entityServiceClient.getCashUnit(Integer.parseInt(cus.getCashUnitId()));
                return cus.getCashUnitStockQuantity() * cashUnit.getCashUnitValueAmount();
            }).sum();

        // 7. Validate the computed amount with the registered CashStockOperationAmount
        if (!cashStock.getCashStockAmount().equals(totalCashValue)) {
            throw new IllegalArgumentException("CashStockOperationAmount does not match the total calculated value");
        }

        return true;
    }
}
