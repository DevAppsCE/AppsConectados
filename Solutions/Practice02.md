# Prática 2 - Countries para iOS
Nessa prática vamos modificar o projeto inicial do aplicativo "Countries" para iOS, para que ele obtenha as informações de países a partir da API do serviço [Geonames.org](http://www.geonames.org).

## Exercício 1 - Obtendo dados do Geonames

**1.** Baixe o [código inicial](../Practices/Countries-iOS-Starter.zip) do projeto, descompacte em uma pasta de trabalho e abra o projeto Countries no Xcode.

**2.** Execute o projeto e verifique seu funcionamento. A listagem da tela (`CountriesViewController.swift`) inicial contém 5 países, inseridos manualmente no código do projeto. Ao selecionar um país você é levado para a tela de detalhes do país (`CountryDetailsViewController.swift`). Note também que as imagens das bandeiras dos países não esta sendo carregada, mostrando apenas um _placeholder_. Toque no botão refresh na barra de navegação a direita, uma animação de progresso será exibida por alguns instantes, seguida de um erro.

**3.** O download da lista de países é feito pelo modelo. Abra o arquivo `CountryList.swift` na pasta `Models`, responsável por gerenciar a listagem de países. Localize o método `refreshListFromGeonamesService:` e veja sua implementação padrão. Nela geramos um atraso (_delay_) artificial de 1,5 segundos e retornamos o _callback_ informando falha na operação.

**4.** Antes de acessar o API do Geonames, precisamos primeiro determinar o endereço onde as chamadas serão feitas. Para isso vamos criar uma constante no arquivo `CountryList.swift` na seção _Properties_. Inclua o código abaixo:

```swift
static let COUNTRY_INFO_URL = "http://api.geonames.org/countryInfoJSON?username=ravero"
```

**5.** A seguir, vamos construir o código básico para obter dados através da biblioteca `URLSession`. Localize o método `refreshListFromGeonamesService` e substitua sua implementação pelo código abaixo:

```swift
let serviceUrl = URL(string: CountryList.COUNTRY_INFO_URL)!
let downloadDataTask = URLSession.shared.dataTask(with: serviceUrl, completionHandler:
{ (data, response, error) -> Void in
    // Check if there is error
    if let e = error {
        print("Error trying to download data: \(e)")
        return
    }
})

// Start the download
downloadDataTask.resume()
```

Teste o código acima executando o programa e tentando obter os dados de países a partir do botão _Refresh_. Verifique o Console do Xcode, e note que um erro será apresentado. Note também que a animação de progresso continua executando indefinidamente.

**6.** A falha acontece por conta do `ATS` (App Transport Security), recurso incluído pela Apple no iOS que permite que sejam feitas somente conexões seguras com servidores https. Precisamos desabilitar esse recurso para poder integrar com o Geonames, que não implementa o SSL. Localize o arquivo `Info.plist`, clique com o botão direito e selecione **_Open As > Source Code_** no menu de contexto. Inclua o código abaixo antes das tags de fechamento `</dict></plist>`:

```swift
<key>NSAppTransportSecurity</key>
<dict>
    <key>NSAllowsArbitraryLoads</key>
    <true/>
</dict>
```

**7.** Ao executar de novo o app, isso corrigirá o problema do download, porém continuamos com a animação inifinita. Isso acontece pois não chamamos a função de _callback_ para informar que a operação assíncrona foi concluída. Substitua a implementação do método `refreshListFromGeonamesService` pelo código abaixo, que além de incluir essas chamadas também imprime o conteúdo retornado pelo Geonames no Console do Xcode:

```swift
let serviceUrl = URL(string: CountryList.COUNTRY_INFO_URL)!
let downloadDataTask = URLSession.shared.dataTask(with: serviceUrl, completionHandler:
{ (data, response, error) -> Void in
    // Check if there is error
    if let e = error {
        print("Error trying to download data: \(e)")
        if let c = completionHandler {
            DispatchQueue.main.async {
                c(false)
            }
        }
        return
    }
    
    let content = String(data: data!, encoding: String.Encoding.utf8)!
    print("Response body: \(content)")

    if let c = completionHandler {
        DispatchQueue.main.async {
            c(true)
        }
    }
})

// Start the download
downloadDataTask.resume()
```

**7.** Agora que já vimos que os dados estão sendo baixados corretamente, vamos implementar o código final que interpreta o JSON retornado e atualiza a listagem de países. Primeiro precisamos fazer uma pequena atualização na constante com o endereço da URL, para incluir o parâmetro que determina em que língua as informações de países devem ser retornadas. Modifique o código conforme a seguir:

```swift
static let COUNTRY_INFO_URL = "http://api.geonames.org/countryInfoJSON?{lang}username=ravero"
```

**8.** A implementação final do método de refresh segue conforme abaixo:

```swift
func refreshListFromGeonamesService(language localeCode: String? = nil, _ completionHandler: ((_ success: Bool) -> ())?) {
    // Configura um Objeto URL com o caminho para o endpoint do Geonames que vamos consultar para obter a lista de países.
    // Usamos uma substituição simples para incluir o parâmetro de codigo da língua em que os dados devem ser retornados.    
    let serviceUrl = URL(string: CountryList.COUNTRY_INFO_URL.replacingOccurrences(of: "{lang}", with: getLangParam(languageCode: localeCode)))!
    
    let downloadDataTask = URLSession.shared.dataTask(with: serviceUrl, completionHandler:
        { (data, response, error) -> Void in
            // Check if there is error
            if let e = error {
                print("Error trying to download data: \(e)")
                if let c = completionHandler {
                    DispatchQueue.main.async {
                        c(false)
                    }
                }
                
                return
            }
            
            // Processo o conteúdo JSON retornado pelo serviço
            self.parseGeonamesCountryInfoJsonData(data!)
            if let c = completionHandler {
                DispatchQueue.main.async {
                    c(true)
                }
            }
    })
    
    // Start the download
    downloadDataTask.resume()
}
```

**9.** A implementação do código anterior depende de dois novos métodos ainda não existentes. O primeiro é responsável por substituir o parâmetro de língua na URL do Geonames. Inclua o método na classe `CountryList`:

```swift
private func getLangParam(languageCode: String?) -> String {
    if let languageCode = languageCode {
        return "lang=\(languageCode)&"
    }
    return ""
}
```

**10.** O segundo método é responsável por interpretar o código JSON recebido o Geonames, e transformá-lo em objetos que irão alimentar a lista de países mantida pela instância da classe `CountryList`. Como nele faremos uso da biblioteca `SwiftyJSON` para facilitar a extração do conteúdo, inclua a cláusula `import` no início do arquivo, logo abaixo da importação da biblioteca `UIKit`:

```swift
import SwiftyJSON
```

E inclua a implementação do método `parseGeonamesCountryInfoJsonData` conforme abaixo:

```swift
/**
    Recebe uma lista de países como uma String JSON, interpreta e alimenta a lista mantida por esse objeto.
 */
private func parseGeonamesCountryInfoJsonData(_ data: Data) {
    guard let json = try? JSON(data: data) else {
        return
    }
    let items = json["geonames"].arrayValue
    self.countries.removeAll(keepingCapacity: false)

    for item in items {
        let countryName = item["countryName"].stringValue
        let countryCode = item["countryCode"].stringValue
        let capital = item["capital"].stringValue
        let continent = item["continent"].stringValue
        let population = item["population"].intValue
        let area = item["areaInSqKm"].floatValue        

        self.countries.append(Country(code: countryCode, name: countryName, capital: capital, continent: continent, population: population, area: area))
    }
}
```

Execute e teste o aplicativo. Com isso incluímos a integração do serviço Geonames em nosso app.


## Exercício 2 - Obtendo as Bandeiras

**1.** No arquivo `CountryViewCell.swift` inclua a instrução `import` da biblioteca `SDWebImage` no início do arquivo:

```swift
import SDWebImage
```

**2.** E dentro do `if` no código do `didSet` da propriedade `country`, inclua o código abaixo para que a biblioteca baixe a bandeira a partir da URL construída pelo propriedade `flagUrl` da classe `Country`:

```swift
flagImageView.sd_setImage(with: country.flagUrl, placeholderImage: UIImage(named: "UnknownFlag")!)
```

**3.** Repita o processo para o arquivo `CountryDetailsViewController.swift`, responsável por carregar as informações na tela de detalhes. Inclua a instrução `import` no início do arquivo: 

```swift
import SDWebImage
```

**4.** E dentro do método `viewDidLoad` localize o `if` e coloque as seguintes instruções no final do bloco:

```swift
// Load flag image aside
let unknownImage = UIImage(named: "UnknownFlag")
flagImageView.sd_setImage(with: self.country.flagUrl as URL, placeholderImage: unknownImage)
```

Execute o aplicativo e verifique se as bandeiras estão sendo baixadas corretamente tanto na listagem quanto na tela de detalhes.