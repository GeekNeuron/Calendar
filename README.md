@echo off
chcp 65001 > nul

echo.
echo ====================================================================
echo      IN SCRIPT TANZIMATE PROXY, DNS VA SHABAKE RA RESET MIKONAD
echo ====================================================================
echo.
echo Baraye ejra, bayad dastresie Administrator dashte bashid.
echo.
pause
cls

echo [1/3] Dar hale reset kardan tanzimate Proxy...
:: Non-aktif kardan proxy
reg add "HKCU\Software\Microsoft\Windows\CurrentVersion\Internet Settings" /v ProxyEnable /t REG_DWORD /d 0 /f
:: Pak kardan adres proxy server
reg delete "HKCU\Software\Microsoft\Windows\CurrentVersion\Internet Settings" /v ProxyServer /f > nul 2>&1
echo Tanzimate Proxy reset shod.
echo.

echo [2/3] Dar hale reset kardan Winsock va TCP/IP...
netsh winsock reset
netsh int ip reset
echo Winsock va TCP/IP reset shod.
echo.

echo [3/3] Dar hale pak kardan cache DNS va gereftan IP jadid...
ipconfig /release
ipconfig /renew
ipconfig /flushdns
echo Cache DNS pak shod.
echo.

echo ====================================================================
echo      *** HAME MOارد با موفقیت انجام شد ***
echo ====================================================================
echo.
echo !! Mohem: Baraye tatbigh kamel taghirat, lotfan system khod ra restart konid. !!
echo.
pause
