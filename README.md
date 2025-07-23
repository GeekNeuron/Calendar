@echo off
chcp 65001 > nul
title Python Local Package Installer

echo =================================================================
echo    Installer baraye package-haye Python dar hamin pooshe
echo =================================================================
echo.
echo Press any key to start the installation...
pause > nul
cls

echo.
echo [+] Dar hale barresie package-haye .whl ...
FOR %%f IN (*.whl) DO (
    echo [*] Nasbe package: %%f
    pip install "%%f"
)

echo.
echo [+] Dar hale barresie package-haye .tar.gz ...
FOR %%f IN (*.tar.gz) DO (
    echo [*] Nasbe package: %%f
    pip install "%%f"
)

echo.
echo =================================================================
echo    Tamame package-ha barresi va nasb shodand.
echo =================================================================
echo.
echo Press any key to exit...
pause > nul
