
# NuGet restore															
 FROM mcr.microsoft.com/dotnet/core/sdk:3.0 AS build						
 WORKDIR /src 																
 COPY *.sln . 																
 COPY MyOcelot.UnitTests/*.csproj MyOcelot.UnitTests/ 	
 COPY MyOcelot/*.csproj MyOcelot/ 						
 RUN dotnet restore 														
 COPY . . 																	
  																			
 # testing 																
 FROM build AS testing 													
 WORKDIR /src/MyOcelot										
 RUN dotnet build 															
 WORKDIR /src/MyOcelot.UnitTests 								
 RUN dotnet test 															
  																			
 # publish 																
 FROM build AS publish 													
 WORKDIR /src/MyOcelot 											
 RUN dotnet publish -c Release -o /src/publish 							
 																			
 FROM mcr.microsoft.com/dotnet/core/aspnet:3.0 AS runtime 					
 WORKDIR /app 																
 COPY --from=publish /src/publish . 										
 # ENTRYPOINT ["dotnet", "MyOcelot.dll"] 						
 # heroku uses the following 												
 CMD ASPNETCORE_URLS=http://*:$PORT dotnet MyOcelot.dll 
