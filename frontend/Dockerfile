FROM mcr.microsoft.com/dotnet/aspnet:6.0
WORKDIR /app
EXPOSE 80
EXPOSE 443
COPY frontend-app .
ENTRYPOINT ["dotnet", "frontend.dll"]
