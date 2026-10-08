# ============================================================
# Client Ledger - Order Load Test
# ============================================================
# API:
#   POST http://localhost:8081/api/orders
#
# Test:
#   6 months
#   10 orders per day
#   Multiple client IDs
#   1 / 5 / 10 / 15 items per order
#   Sequential API calls
#   Bearer token on every request
#   Response-time statistics
# ============================================================

$BaseUrl = "http://localhost:8081"

# ------------------------------------------------------------
# PUT YOUR FIREBASE ID TOKEN HERE
# ------------------------------------------------------------
$Token = "PASTE_YOUR_FIREBASE_ID_TOKEN_HERE"

# ------------------------------------------------------------
# PUT YOUR CLIENT IDS HERE
# The script will rotate through these clients.
# ------------------------------------------------------------
$ClientIds = @(
    "CLI-MUXZMK5L"
    # "CLI-XXXXXXXXX"
    # "CLI-YYYYYYYYY"
)

# ------------------------------------------------------------
# TEST CONFIGURATION
# ------------------------------------------------------------
$Months = 6
$OrdersPerDay = 10

# Set to $true if you want to see every request.
$VerboseRequests = $false

# ------------------------------------------------------------
# VALIDATION
# ------------------------------------------------------------
if ($Token -eq "PASTE_YOUR_FIREBASE_ID_TOKEN_HERE" -or
    [string]::IsNullOrWhiteSpace($Token)) {

    Write-Host ""
    Write-Host "ERROR: Please put your Firebase ID token in `$Token." -ForegroundColor Red
    exit 1
}

if ($ClientIds.Count -eq 0) {
    Write-Host ""
    Write-Host "ERROR: Please provide at least one client ID." -ForegroundColor Red
    exit 1
}

# ------------------------------------------------------------
# HTTP HEADERS
# ------------------------------------------------------------
$Headers = @{
    "Authorization" = "Bearer $Token"
    "Content-Type"  = "application/json"
    "Accept"        = "application/json"
}

# ------------------------------------------------------------
# DATE RANGE
# ------------------------------------------------------------
$EndDate = (Get-Date).Date
$StartDate = $EndDate.AddMonths(-$Months)

Write-Host ""
Write-Host "============================================================"
Write-Host " CLIENT LEDGER ORDER LOAD TEST"
Write-Host "============================================================"
Write-Host "API              : $BaseUrl/api/orders"
Write-Host "Start Date       : $($StartDate.ToString('yyyy-MM-dd'))"
Write-Host "End Date         : $($EndDate.ToString('yyyy-MM-dd'))"
Write-Host "Months           : $Months"
Write-Host "Orders / Day     : $OrdersPerDay"
Write-Host "Client Count     : $($ClientIds.Count)"
Write-Host "Expected Orders  : approximately $([int](($EndDate - $StartDate).TotalDays + 1) * $OrdersPerDay)"
Write-Host "============================================================"
Write-Host ""

# ------------------------------------------------------------
# ITEM DATA
# ------------------------------------------------------------

$ProductNames = @(
    "Product-A",
    "Product-B",
    "Product-C",
    "Product-D",
    "Product-E",
    "Product-F",
    "Product-G",
    "Product-H",
    "Product-I",
    "Product-J",
    "Product-K",
    "Product-L",
    "Product-M",
    "Product-N",
    "Product-O"
)

$Units = @(
    "SINGLE",
    "DOZEN",
    "BOX"
)

# ------------------------------------------------------------
# STATISTICS
# ------------------------------------------------------------

$Successful = 0
$Failed = 0

$ResponseTimes = New-Object System.Collections.Generic.List[double]

$FailureDetails = New-Object System.Collections.Generic.List[object]

$TotalStopwatch = [System.Diagnostics.Stopwatch]::StartNew()

$orderNumber = 0
$clientIndex = 0

# ------------------------------------------------------------
# GENERATE RANDOM ITEMS
# ------------------------------------------------------------

function New-OrderItems {

    # Required distribution:
    # 1 item
    # 5 items
    # 10 items
    # 15 items

    $itemCounts = @(1, 5, 10, 15)

    $itemCount = Get-Random -InputObject $itemCounts

    $items = @()

    for ($i = 0; $i -lt $itemCount; $i++) {

        $productName = $ProductNames[$i % $ProductNames.Count]

        $quantity = Get-Random -Minimum 1 -Maximum 11

        $unit = Get-Random -InputObject $Units

        # Amount per unit/item.
        # Keep values reasonable for testing.
        $amount = Get-Random -Minimum 500 -Maximum 10001

        # Use common GST rates.
        $gstRate = Get-Random -InputObject @(0, 5, 12, 18)

        # Controller expects gstAmount from the request.
        $gstAmount = [math]::Round(
            ($amount * $quantity * $gstRate) / 100
        )

        # Random expense.
        $expense = Get-Random -Minimum 0 -Maximum 1001

        $items += @{
            name      = "$productName-$($i + 1)"
            quantity  = $quantity
            unit      = $unit
            amount    = $amount
            gstRate   = $gstRate
            gstAmount = $gstAmount
            expense   = $expense
        }
    }

    return $items
}

# ------------------------------------------------------------
# CREATE ORDERS
# ------------------------------------------------------------

$currentDate = $StartDate

while ($currentDate -le $EndDate) {

    Write-Host ""
    Write-Host "Processing date: $($currentDate.ToString('yyyy-MM-dd'))"

    for ($dayOrder = 1; $dayOrder -le $OrdersPerDay; $dayOrder++) {

        $orderNumber++

        # ----------------------------------------------------
        # Rotate clients
        # ----------------------------------------------------

        $clientId = $ClientIds[$clientIndex % $ClientIds.Count]

        $clientIndex++

        # ----------------------------------------------------
        # Generate items
        # ----------------------------------------------------

        $items = New-OrderItems

        # ----------------------------------------------------
        # Delivery date
        # ----------------------------------------------------

        $deliveryDate = $currentDate.AddDays(
            (Get-Random -Minimum 1 -Maximum 8)
        )

        # ----------------------------------------------------
        # Request body
        # ----------------------------------------------------

        $body = @{
            clientId     = $clientId
            orderDate    = $currentDate.ToString("yyyy-MM-dd")
            deliveryDate = $deliveryDate.ToString("yyyy-MM-dd")
            items        = $items
        } | ConvertTo-Json -Depth 10

        # ----------------------------------------------------
        # API CALL + TIMING
        # ----------------------------------------------------

        $stopwatch = [System.Diagnostics.Stopwatch]::StartNew()

        try {

            $response = Invoke-WebRequest `
                -Uri "$BaseUrl/api/orders" `
                -Method POST `
                -Headers $Headers `
                -Body $body `
                -UseBasicParsing `
                -ErrorAction Stop

            $stopwatch.Stop()

            $elapsedMs = $stopwatch.Elapsed.TotalMilliseconds

            $ResponseTimes.Add($elapsedMs)

            $Successful++

            if ($VerboseRequests) {
                Write-Host (
                    "  Order {0,-5} | Client {1,-15} | Items {2,-2} | {3,8:N2} ms | HTTP {4}" -f
                    $orderNumber,
                    $clientId,
                    $items.Count,
                    $elapsedMs,
                    $response.StatusCode
                ) -ForegroundColor Green
            }
            else {
                Write-Host "." -NoNewline
            }

        }
        catch {

            $stopwatch.Stop()

            $elapsedMs = $stopwatch.Elapsed.TotalMilliseconds

            $ResponseTimes.Add($elapsedMs)

            $Failed++

            $errorMessage = $_.Exception.Message

            $statusCode = "N/A"

            if ($_.Exception.Response) {
                try {
                    $statusCode = [int]$_.Exception.Response.StatusCode
                }
                catch {
                    $statusCode = "N/A"
                }
            }

            $FailureDetails.Add(
                [PSCustomObject]@{
                    OrderNumber = $orderNumber
                    Date        = $currentDate.ToString("yyyy-MM-dd")
                    ClientId    = $clientId
                    ItemCount   = $items.Count
                    StatusCode  = $statusCode
                    Error       = $errorMessage
                }
            )

            Write-Host ""
            Write-Host (
                "  FAILED Order {0} | Client {1} | Items {2} | {3} ms | HTTP {4}" -f
                $orderNumber,
                $clientId,
                $items.Count,
                [math]::Round($elapsedMs, 2),
                $statusCode
            ) -ForegroundColor Red
        }
    }

    $currentDate = $currentDate.AddDays(1)
}

$TotalStopwatch.Stop()

# ------------------------------------------------------------
# PERFORMANCE CALCULATION
# ------------------------------------------------------------

Write-Host ""
Write-Host ""
Write-Host "============================================================"
Write-Host " LOAD TEST COMPLETE"
Write-Host "============================================================"

$TotalOrders = $Successful + $Failed

Write-Host "Total Orders     : $TotalOrders"
Write-Host "Successful       : $Successful"
Write-Host "Failed           : $Failed"
Write-Host "Total Time       : $([math]::Round($TotalStopwatch.Elapsed.TotalSeconds, 2)) seconds"

if ($TotalOrders -gt 0) {

    $Average = ($ResponseTimes | Measure-Object -Average).Average
    $Minimum = ($ResponseTimes | Measure-Object -Minimum).Minimum
    $Maximum = ($ResponseTimes | Measure-Object -Maximum).Maximum

    $SortedTimes = $ResponseTimes | Sort-Object

    $p50Index = [math]::Ceiling($SortedTimes.Count * 0.50) - 1
    $p95Index = [math]::Ceiling($SortedTimes.Count * 0.95) - 1
    $p99Index = [math]::Ceiling($SortedTimes.Count * 0.99) - 1

    $P50 = $SortedTimes[[math]::Max(0, $p50Index)]
    $P95 = $SortedTimes[[math]::Max(0, $p95Index)]
    $P99 = $SortedTimes[[math]::Max(0, $p99Index)]

    Write-Host ""
    Write-Host "------------------------------------------------------------"
    Write-Host " RESPONSE TIME"
    Write-Host "------------------------------------------------------------"
    Write-Host "Minimum          : $([math]::Round($Minimum, 2)) ms"
    Write-Host "Average          : $([math]::Round($Average, 2)) ms"
    Write-Host "P50              : $([math]::Round($P50, 2)) ms"
    Write-Host "P95              : $([math]::Round($P95, 2)) ms"
    Write-Host "P99              : $([math]::Round($P99, 2)) ms"
    Write-Host "Maximum          : $([math]::Round($Maximum, 2)) ms"
    Write-Host "------------------------------------------------------------"
}

# ------------------------------------------------------------
# FAILURE SUMMARY
# ------------------------------------------------------------

if ($FailureDetails.Count -gt 0) {

    Write-Host ""
    Write-Host "============================================================"
    Write-Host " FAILED REQUESTS"
    Write-Host "============================================================"

    $FailureDetails | Format-Table -AutoSize
}

Write-Host ""
Write-Host "============================================================"
Write-Host " DONE"
Write-Host "============================================================"
