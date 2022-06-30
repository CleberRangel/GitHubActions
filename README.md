# Building MAUI Application with GitHub Actions

In this repo we have the sample application for the .NET MAUI for `Windows` desktop and we test and build it, and release it under the Releases in the GitHub.

The content of this repo came from the following sources:

[Gerald Versluis](https://www.youtube.com/watch?v=8lvdLa0v8zY&ab_channel=GeraldVersluis)

[Publish a .NET MAUI app for Windows](https://docs.microsoft.com/en-us/dotnet/maui/windows/deployment/overview)

[Create a certificate for package signing](https://docs.microsoft.com/en-us/windows/msix/package/create-certificate-package-signing)

## Creating Self Signed Certificate
For the application to be able to be installed we need to create a Signed Certificate, if we were deplying it to the Microsoft Store they would create a certificate, but this is not this case.

Follow the steps from:

[Create a certificate for package signing](https://docs.microsoft.com/en-us/windows/msix/package/create-certificate-package-signing)

### Consider the following

- Create a certificate

Notice that `"Contoso Software, O=Contoso Corporation, C=US"` needs to match with the same value inside the `Package.appxmanifest` on `Platforms > Windows` of your .NET MAUI project.

![CN=User Name](/images/CN%3DUser%20Name.png)

Save the Thumprint value created.

- Export PFX certificate

User the `Password usage` approach

## GitHub Secrets

Since we don't want to push out certificate with the solution, this would be a real bad practice...

We need to use the GitHub Actions Secrets, access them from your Repo Settings.

![GitHub Secrets](/images/AcessGitHub_Secrets.png)

We cannot upload files for the Secrets, but we can convert our PFX cert to file.

From PowerShell do the following:

```PowerShell
certutil.exe -encode <CERT PATH> <OUTPUT PATH FILE>
```

You can open the output file and create a Secret variable with the content.

You also need of Secret with the `certificate thumbprint` and the `certificate password`.


## Explaining build-windows.yml

We have a few enviroment variables to facilite our life:

```YML
env:
  ARTIFACT_NAME: MAUI.Installer
  PROJECT_PATH: src\MauiWithGitActions\MauiWithGitActions.csproj
  PFX_ASC_FILE: cert.pfx.asc
  PFX_FILE: cert.pfx

```

The most intersting part of the Build job is the following:

We create a certificate from the Secret ASCII data.

```YML
- name: Decrypt PFX Certificate file
        run: | 
            echo "${{secrets.WINDOWS_PFX_FILE}}" > ${{env.PFX_ASC_FILE}}
            certutil -decode ${{env.PFX_ASC_FILE}} ${{env.PFX_FILE}}
```

We add the Certificate to the windows running machine:

```YML
  - name: Add Cert to Windows Host
        run: certutil -user -q -p ${{secrets.WINDOWS_PFX_PASSWORD}} -importpfx ${{env.PFX_FILE}} NoRoot
```

Build our apllication MSIX install file:

```YML
- name: Publish app for release
        run: dotnet publish ${{env.PROJECT_PATH}} -c Release -f:net6.0-windows10.0.19041.0 /p:GenerateAppxPackageOnBuild=true /p:AppxPackageSigningEnabled=true /p:PackageCertificateThumbprint="${{secrets.WINDOWS_PFX_PRINT}}"
```

And upload the MSIX file to our repo Release page

```YML
      - name: Create Release
        uses: ncipollo/release-action@v1
        with:
          artifacts: .\**\AppPackages\**\MauiWithGitActions*.msix
          allowUpdates: true
          tag: ${{ github.ref }}
          token: ${{ secrets.GITHUB_TOKEN }}
```


![GitHub Release Page](/images/GitHub_Release_Page.png)

The user can download the MauiWithGitActions file to install, however there a few things they neeed to do to be able to install it.

Reference to the `PUBLISH` part of the article: [Publish a .NET MAUI app for Windows](https://docs.microsoft.com/en-us/dotnet/maui/windows/deployment/overview)