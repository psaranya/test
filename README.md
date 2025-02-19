 public boolean validateOpenCashStock(CashStocksActivateContextRequest request) throws CashException {
        // 1. Validate CashStockId exists
        CashStock cashStock = entityServiceClient.getCashStock(Integer.parseInt(request.getCashStockId()));
        if(cashStock == null) {
            throw new CashException("KASH_10000001","CashStockId must exist", HttpStatus.valueOf(String.valueOf(HttpStatus.BAD_REQUEST)),"error","");
        }

        // 2. Validate CashStockStatusName is "CLOSE"
        if (!"CLOSE".equals(cashStock.getCashStockStatusName())) {
            throw new CashException("KASH_10000002","CashStockStatusName of the CashStockId provided must be “CLOSE”.", HttpStatus.valueOf(String.valueOf(HttpStatus.BAD_REQUEST)),"error","");
        }

        // 3. Validate CashUnitId list
        List<CashUnit> cashUnitList = request.getCashUnitStockContextList().stream()
                .map(src -> {
                    CashUnit cashUnit = entityServiceClient.getCashUnit(Integer.parseInt(src.getCashUnitId()));
                    if(cashUnit == null) {
                        try {
                            throw new CashException("KASH_10000003","CashUnitId does not exist", HttpStatus.valueOf(String.valueOf(HttpStatus.BAD_REQUEST)),"error","");
                        } catch (CashException e) {
                            throw new RuntimeException(e);
                        }
                    }
                    return cashUnit;
                }).collect(Collectors.toList());


        //  Validate CashStockSession exists
        CashStockSession cashStockSession = entityServiceClient.getCashStockSession(
                       cashStock.getCashStockLastSessionId());
        if(cashStockSession == null) {
             throw new CashException("KASH_10000004","CashStockLastSessionId does not exist", HttpStatus.valueOf(String.valueOf(HttpStatus.BAD_REQUEST)),"error","");
        }

        //  Compute total cash value from provided CashUnitStocks
        int totalCashValue = request.getCashUnitStockContextList().stream()
                .mapToInt(cus -> {
                    CashUnit cashUnit = entityServiceClient.getCashUnit(Integer.parseInt(cus.getCashUnitId()));
                    return cus.getCashUnitStockQuantity() * cashUnit.getCashUnitValueAmount();
                }).sum();

        //  Validate the computed amount with the requested CashStockOperationAmount
        if (!request.getCashStockOperationAmount().equals(totalCashValue*100)) {
            throw new CashException("KASH_10000005","CashStockOperationAmount does not match the total calculated value", HttpStatus.valueOf(String.valueOf(HttpStatus.BAD_REQUEST)),"error","");
        }

        CashStockSession lastCashStockSession = entityServiceClient.getCashStockSession(cashStock.getCashStockLastSessionId());
        if (lastCashStockSession != null && lastCashStockSession.getCashStockSessionStatusName().equals("OPEN")) {
            throw new CashException("KASH_10000006","CashStockLastSessionStatusName is OPEN", HttpStatus.valueOf(String.valueOf(HttpStatus.BAD_REQUEST)),"error","");

        }
        return true;
    }
}





public enum CashErrorCode {
    CASH_STOCK_ID_NOT_FOUND("KASH_10000001", "CashStockId must exist"),
    CASH_STOCK_STATUS_NOT_CLOSE("KASH_10000002", "CashStockStatusName of the CashStockId provided must be CLOSE"),
    CASH_UNIT_ID_NOT_FOUND("KASH_10000003", "CashUnitId does not exist"),
    CASH_STOCK_SESSION_NOT_FOUND("KASH_10000004", "CashStockLastSessionId does not exist"),
    CASH_OPERATION_AMOUNT_MISMATCH("KASH_10000005", "CashStockOperationAmount does not match the total calculated value"),
    CASH_STOCK_SESSION_OPEN("KASH_10000006", "CashStockLastSessionStatusName is OPEN");

    private final String code;
    private final String message;

    CashErrorCode(String code, String message) {
        this.code = code;
        this.message = message;
    }

    public String getCode() {
        return code;
    }

    public String getMessage() {
        return message;
    }
}


public boolean validateOpenCashStock(CashStocksActivateContextRequest request) throws CashException {
    // Validate CashStockId exists
    CashStock cashStock = entityServiceClient.getCashStock(Integer.parseInt(request.getCashStockId()));
    if (cashStock == null) {
        throw new CashException(CashErrorCode.CASH_STOCK_ID_NOT_FOUND.getCode(),
                CashErrorCode.CASH_STOCK_ID_NOT_FOUND.getMessage(),
                HttpStatus.BAD_REQUEST, "error", "");
    }

    // Validate CashStockStatusName is "CLOSE"
    if (!"CLOSE".equals(cashStock.getCashStockStatusName())) {
        throw new CashException(CashErrorCode.CASH_STOCK_STATUS_NOT_CLOSE.getCode(),
                CashErrorCode.CASH_STOCK_STATUS_NOT_CLOSE.getMessage(),
                HttpStatus.BAD_REQUEST, "error", "");
    }

    // Validate CashUnitId list
    List<CashUnit> cashUnitList = request.getCashUnitStockContextList().stream()
            .map(src -> {
                CashUnit cashUnit = entityServiceClient.getCashUnit(Integer.parseInt(src.getCashUnitId()));
                if (cashUnit == null) {
                    throw new CashException(CashErrorCode.CASH_UNIT_ID_NOT_FOUND.getCode(),
                            CashErrorCode.CASH_UNIT_ID_NOT_FOUND.getMessage(),
                            HttpStatus.BAD_REQUEST, "error", "");
                }
                return cashUnit;
            }).collect(Collectors.toList());

    // Validate CashStockSession exists
    CashStockSession cashStockSession = entityServiceClient.getCashStockSession(cashStock.getCashStockLastSessionId());
    if (cashStockSession == null) {
        throw new CashException(CashErrorCode.CASH_STOCK_SESSION_NOT_FOUND.getCode(),
                CashErrorCode.CASH_STOCK_SESSION_NOT_FOUND.getMessage(),
                HttpStatus.BAD_REQUEST, "error", "");
    }

    // Compute total cash value from provided CashUnitStocks
    int totalCashValue = request.getCashUnitStockContextList().stream()
            .mapToInt(cus -> {
                CashUnit cashUnit = entityServiceClient.getCashUnit(Integer.parseInt(cus.getCashUnitId()));
                return cus.getCashUnitStockQuantity() * cashUnit.getCashUnitValueAmount();
            }).sum();

    // Validate the computed amount with the requested CashStockOperationAmount
    if (!request.getCashStockOperationAmount().equals(totalCashValue * 100)) {
        throw new CashException(CashErrorCode.CASH_OPERATION_AMOUNT_MISMATCH.getCode(),
                CashErrorCode.CASH_OPERATION_AMOUNT_MISMATCH.getMessage(),
                HttpStatus.BAD_REQUEST, "error", "");
    }

    // Validate CashStockSession status
    if ("OPEN".equals(cashStockSession.getCashStockSessionStatusName())) {
        throw new CashException(CashErrorCode.CASH_STOCK_SESSION_OPEN.getCode(),
                CashErrorCode.CASH_STOCK_SESSION_OPEN.getMessage(),
                HttpStatus.BAD_REQUEST, "error", "");
    }

    return true;
}

