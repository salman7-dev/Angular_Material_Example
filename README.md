
catch {

    $stopwatch.Stop()

    $elapsedMs = $stopwatch.Elapsed.TotalMilliseconds

    $Failed++

    $statusCode = "N/A"
    $responseBody = ""

    if ($_.Exception.Response) {

        try {
            $statusCode = [int]$_.Exception.Response.StatusCode
        }
        catch {
            $statusCode = "N/A"
        }

        try {
            $reader = New-Object System.IO.StreamReader(
                $_.Exception.Response.GetResponseStream()
            )

            $responseBody = $reader.ReadToEnd()
            $reader.Close()
        }
        catch {
            $responseBody = $_.Exception.Message
        }
    }
    else {
        $responseBody = $_.Exception.Message
    }

    $FailureDetails.Add(
        [PSCustomObject]@{
            OrderNumber = $orderNumber
            Date        = $currentDate.ToString("yyyy-MM-dd")
            ClientId    = $clientId
            ItemCount   = $items.Count
            StatusCode  = $statusCode
            Error       = $responseBody
        }
    )

    Write-Host ""
    Write-Host "FAILED ORDER" -ForegroundColor Red
    Write-Host "  Order       : $orderNumber"
    Write-Host "  Date        : $($currentDate.ToString('yyyy-MM-dd'))"
    Write-Host "  Client      : $clientId"
    Write-Host "  Items       : $($items.Count)"
    Write-Host "  HTTP Status : $statusCode"
    Write-Host "  Response    : $responseBody" -ForegroundColor Yellow
    Write-Host ""
}
