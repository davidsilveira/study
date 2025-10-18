# Hotwire Native — Guia de Estudos (Leitura Técnica Simplificada)

Este guia resume os tópicos principais de Hotwire Native com linguagem simples, exemplos práticos e observações alinhadas à documentação oficial do Hotwire (Turbo + Turbo Native).

Sumário:
- Prefácio
- Construa seus primeiros apps Hotwire Native
- Controle seus apps com Rails
- Navegue com elegância usando Path Configuration
- Adicione uma Tab Bar nativa
- Renderize telas nativas com SwiftUI (iOS)
- Renderize telas nativas com Jetpack Compose (Android)
- Bridge Components (iOS e Android)
- Deploy (TestFlight e Play Console)
- Push Notifications (APNs e FCM)

---

## Preface

### The Problem with Native Apps
Problema clássico: manter dois apps (iOS e Android) com linguagens, ferramentas e ciclos de release diferentes. Isso gera:
- Código duplicado e lógica de negócio repetida
- Custos maiores de desenvolvimento e QA
- Diferenças de UX e timing de features entre plataformas

### The Hybrid Solution
Hotwire Native (Turbo Native) permite usar suas telas HTML do Rails em apps nativos com WebViews otimizadas. Você mantém:
- Uma base de código Rails para UI/negócio
- Apps iOS/Android que usam Turbo para navegar entre páginas
- Partes 100% nativas quando necessário (telas, navegação, câmeras, mapas)

### Prerequisites
- Rails 7+, Turbo (padrão no Rails 7)
- Xcode (iOS) e Android Studio (Android)
- Noções de HTML/ERB, Stimulus, CSS e REST

### How This Book Is Structured
- Conceitos → Primeiro app → Controle via Rails → Navegação (Path Config) → Tabs → Telas nativas (SwiftUI/Compose) → Bridge Components → Deploy → Push

### Need Help?
- Documentação oficial Turbo/Hotwire: https://turbo.hotwired.dev e https://native.hotwired.dev
- Repositórios: hotwire/turbo-ios e hotwire/turbo-android

---

## Build Your First Hotwire Native Apps excerpt

### Build a Hotwire Native iOS App
Objetivo: iniciar um app iOS que abre sua aplicação Rails via Turbo Native (Turbo iOS).

Passos essenciais:
1) Instalar dependência Turbo iOS (Swift Package Manager no Xcode: https://github.com/hotwired/turbo-ios)
2) Criar um UINavigationController com um Session/VisitableViewController da Turbo iOS

Exemplo mínimo (Swift):
```swift
import UIKit
import Turbo

@main
class AppDelegate: UIResponder, UIApplicationDelegate {
  var window: UIWindow?

  func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
    window = UIWindow(frame: UIScreen.main.bounds)

    let session = Session()
    let nav = UINavigationController()
    window?.rootViewController = nav
    window?.makeKeyAndVisible()

    let url = URL(string: "https://sua-app-rails.com/")!
    let vc = VisitableViewController(url: url)
    session.visit(vc)
    nav.pushViewController(vc, animated: false)
    return true
  }
}
```

### Build a Hotwire Native Android App
Objetivo: iniciar um app Android que abre sua aplicação Rails usando Turbo Android.

Passos essenciais:
1) Adicionar o Turbo Android (Gradle) e o Navigator
2) Criar uma Activity que inicializa o Navigator apontando para sua URL inicial

Exemplo mínimo (Kotlin):
```kotlin
class MainActivity : AppCompatActivity() {
  lateinit var session: Session

  override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    setContentView(R.layout.activity_main)

    session = Session(this)
    val url = Uri.parse("https://sua-app-rails.com/")
    val visitable = TurboVisitFragment.newInstance(url)
    supportFragmentManager.beginTransaction()
      .replace(R.id.container, visitable)
      .commit()
  }
}
```

### What’s Next?
- Enviar metadados via HTML para personalizar barra de navegação
- Configurar apresentação de modais (Path Configuration)
- Misturar telas web e totalmente nativas

---

## Control Your Apps with Rails excerpt

### Dynamically Set a Native Title via HTML
Use `content_for :title` nas views. O Turbo Native lê o título e atualiza a Navigation Bar.
```erb
<%# app/views/articles/show.html.erb %>
<%= content_for :title, @article.title %>
<h1><%= @article.title %></h1>
<p><%= @article.body %></p>
```

### Hide the Navigation Bar with Conditional Ruby
Você pode sinalizar para ocultar a barra via meta tag condicionada.
```erb
<%# app/views/layouts/application.html.erb %>
<% content_for :head do %>
  <% if controller_name == 'photos' && action_name == 'show' %>
    <meta name="turbo-ios-nav" content="hidden">
  <% end %>
<% end %>
```
Observação: frameworks Turbo Native costumam ler meta-instruções do DOM/head para decidir UI.

### Hide the Navigation Bar with CSS
Para páginas específicas, use uma classe e uma CSS var lida pelo cliente nativo.
```erb
<div class="fullscreen">
  ...
</div>
```
```css
.fullscreen { --turbo-native-bar-style: hidden; }
```

### Add Missing Elements
Mostre/oculte elementos para web vs app nativo via helper simples:
```ruby
# app/controllers/application_controller.rb
helper_method :turbo_native_app?

def turbo_native_app?
  request.user_agent&.include?("Turbo Native")
end
```
```erb
<% unless turbo_native_app? %>
  <nav class="web-only"> ... </nav>
<% end %>
```

### Keep Users Signed In Between App Launches
Use cookies permanentes/sessões padrão Rails. Em geral, no Turbo Native os cookies da WebView persistem entre execuções.
```ruby
# sessions_controller.rb
cookies.permanent.signed[:user_id] = user.id
```

### Access the Camera and Photos on iOS
Crie um “bridge” nativo para abrir a câmera e retornar resultado à página.
```swift
final class CameraBridge: NSObject, UIImagePickerControllerDelegate, UINavigationControllerDelegate {
  weak var presenter: UIViewController?

  func openCamera() {
    let picker = UIImagePickerController()
    picker.sourceType = .camera
    picker.delegate = self
    presenter?.present(picker, animated: true)
  }

  func imagePickerController(_ picker: UIImagePickerController, didFinishPickingMediaWithInfo info: [UIImagePickerController.InfoKey : Any]) {
    // enviar resultado de volta (JS bridge / postMessage)
    picker.dismiss(animated: true)
  }
}
```

### Access the Camera and Photos on Android
```kotlin
class CameraBridge(private val activity: Activity) {
  fun openCamera() {
    val intent = Intent(MediaStore.ACTION_IMAGE_CAPTURE)
    activity.startActivityForResult(intent, 100)
  }
}
```

### What’s Next?
- Path Configuration para modais, pilhas e roteamento
- Tabs nativas

---

## Navigate Gracefully with Path Configuration

### Routing Modals
Você define padrões de URL que devem abrir como modal.
```json
{
  "rules": [
    { "patterns": ["/sessions/new", "/photos/*/edit"], "properties": { "presentation": "modal" } }
  ]
}
```

### Set Up iOS Path Configuration
Adicione o arquivo ao bundle e/ou a uma URL remota.
```swift
let pathConfig = PathConfiguration(sources: [
  .file(Bundle.main.url(forResource: "path-configuration", withExtension: "json")!),
  .server("https://sua-app-rails.com/native/ios/path-configuration.json")
])
Session.shared.pathConfiguration = pathConfig
```

### Wire Up the iOS Client
Ao criar controllers de navegação, o Session/VisitableViewController respeita as regras do path configuration.

### Set Up Android Path Configuration
```kotlin
val config = PathConfiguration(
  context = this,
  location = PathConfiguration.Location(assetFilePath = "json/path_configuration.json")
)
Turbo.config.pathConfiguration = config
```

### Wire Up the Android Client
O Fragment/Activity do Turbo Android usa a configuração para decidir quando usar modal, intents, pilhas, etc.

### What’s Next?
- Tab bar nativa por plataforma

---

## Add a Native Tab Bar

### Clean Up the iOS Directories
Organize pastas: Controllers/, Views/, Bridges/, Config/.

### Configure Tabs on iOS
Crie um UITabBarController e cada aba com UINavigationController apontando para uma URL raiz.
```swift
class TabsController: UITabBarController {
  override func viewDidLoad() {
    super.viewDidLoad()
    let home = UINavigationController()
    home.tabBarItem = UITabBarItem(title: "Home", image: UIImage(systemName: "house"), tag: 0)
    let profile = UINavigationController()
    profile.tabBarItem = UITabBarItem(title: "Perfil", image: UIImage(systemName: "person"), tag: 1)
    viewControllers = [home, profile]

    home.visit(url: URL(string: "https://sua-app-rails.com/home")!)
    profile.visit(url: URL(string: "https://sua-app-rails.com/profile")!)
  }
}
```

### Clean Up the Android Packages
Separe packages por feature: activities/, fragments/, navigation/.

### Configure Tabs on Android
Use BottomNavigationView + NavHostFragment, associando cada item à sua rota/URL.
```xml
<com.google.android.material.bottomnavigation.BottomNavigationView
  android:id="@+id/bottom_nav"
  app:menu="@menu/bottom_nav_menu" />
```

### What’s Next?
- Telas 100% nativas quando o HTML não basta

---

## Render Native Screens with SwiftUI excerpt

### When to Go Native
- Interações muito gráficas (mapas, câmera, gráficos)
- Acesso a hardware/SDKs específicos
- UI/UX que excede a proposta HTML

### Build a Native Screen with SwiftUI
```swift
struct HikeDetailView: View {
  let hike: Hike
  var body: some View {
    VStack(alignment: .leading) {
      Text(hike.name).font(.largeTitle)
      Text(hike.description)
    }.padding()
  }
}
```

### Render a Map with a SwiftUI View
```swift
import MapKit
struct MapSection: View {
  @State var region: MKCoordinateRegion
  var body: some View { Map(coordinateRegion: $region).frame(height: 240) }
}
```

### Connect to Hotwire Native with a View Controller
Use um UIViewController que carrega dados (via JSON do Rails) e hospeda a View SwiftUI.
```swift
let vc = UIHostingController(rootView: HikeDetailView(hike: hike))
nav.pushViewController(vc, animated: true)
```

### Route the URL via Path Configuration
```json
{ "rules": [ { "patterns": ["/hikes/*"], "properties": { "presentation": "push" } } ] }
```

### Expose a JSON Endpoint for Structured Hike Data
```ruby
# app/controllers/api/hikes_controller.rb
render json: { id: @hike.id, name: @hike.name, description: @hike.description }
```

### Fetch and Parse JSON with a Model and a View Model
```swift
struct Hike: Decodable { let id: Int; let name: String; let description: String }
```

### What’s Next?
- Compose para Android com abordagem equivalente

---

## Render Native Screens with Jetpack Compose

### Integrate Jetpack Compose with Hotwire Native
Ative Compose no Gradle (buildFeatures.compose=true) e adicione dependências.

### Build a Jetpack Compose Screen
```kotlin
@Composable
fun HikeDetailScreen(hike: Hike) {
  Column(Modifier.padding(16.dp)) {
    Text(hike.name, style = MaterialTheme.typography.headlineLarge)
    Text(hike.description)
  }
}
```

### Route the URL via Path Configuration
```json
{ "rules": [ { "patterns": ["/hikes/*"], "properties": { "destination": "hike_detail" } } ] }
```

### Configure Google Maps with an API Key
No Manifest: `meta-data com.google.android.geo.API_KEY` e permissões INTERNET/LOCATION.

### Add a Map to the View
```kotlin
GoogleMap(cameraPositionState = rememberCameraPositionState()) {
  Marker(state = MarkerState(LatLng(hike.lat, hike.lng)))
}
```

### Fetch and Parse JSON with Model and View Model
```kotlin
@Serializable data class Hike(val id:Int, val name:String, val description:String)
```

### What’s Next?
- Bridge Components para comunicação Web ↔ Nativo

---

## Build iOS Bridge Components with Swift

### Install Hotwire Native Bridge
Adicione Turbo iOS e implemente um “bridge” simples com postMessage (WKScriptMessageHandler) ou libs auxiliares do ecossistema.

### Add the HTML Markup
```erb
<div data-controller="share" data-bridge-component="share">
  <button data-action="click->share#send">Compartilhar</button>
</div>
```

### Create a Stimulus Controller
```javascript
import { BridgeComponent } from "@hotwired/hotwire-native-bridge"
export default class extends BridgeComponent {
  send() { this.send("share", { text: "Olá" }) }
}
```

### Create a Native Component
```swift
final class ShareBridge: NSObject {
  func handle(message: [String:Any]) { /* abrir UIActivityViewController */ }
}
```

### Customize the Component
- Trate callbacks de sucesso/erro
- Ajuste apresentações (popover iPad)

### What’s Next?
- Repetir padrão no Android

---

## Build Android Bridge Components with Kotlin

### Create a Native Component
```kotlin
class ShareBridge(private val activity: Activity) {
  fun send(text:String){
    val intent = Intent(Intent.ACTION_SEND).apply{
      type = "text/plain"
      putExtra(Intent.EXTRA_TEXT, text)
    }
    activity.startActivity(Intent.createChooser(intent, "Compartilhar"))
  }
}
```

### Respond to Button Taps
No Stimulus, envie mensagens para o bridge; no Android, trate intents e callbacks.

### Remove Duplicate Buttons / Make the Button Text Dynamic / Add a Dynamic Image
Use JS injetado na WebView (evaluateJavascript) para manipular DOM e sincronizar estado com o app nativo.

### What’s Next?
- Preparar publicação

---

## Deploy to Physical Devices with TestFlight and Play Testing

### Add an App to App Store Connect
- Configure Bundle ID, assinaturas e capabilities no Xcode
- Crie o app no App Store Connect

### Archive and Upload a Build
- Xcode → Product → Archive → Distribute → App Store Connect (Upload)

### Download the App on TestFlight
- Ative TestFlight → convide testadores internos/externos

### Add an App to the Play Console
- Crie app, preencha listagens e políticas

### Generate and Upload a Signed App Bundle
- Gere keystore e `bundleRelease` (.aab)

### Distribute Builds to Testers
- Internal/Closed testing e convites

### What’s Next?
- Push notifications

---

## Send Push Notifications with APNs and FCM

### Send Push Notifications from Ruby on Rails
Mantenha DeviceTokens por usuário e serviço para envio (APNs/FCM).
```ruby
class DeviceToken < ApplicationRecord
  belongs_to :user
  validates :token, presence: true
  validates :platform, inclusion: { in: %w[ios android] }
end
```

### Configure iOS for Push Notifications
- Habilite Push Notifications & Background Modes
- Registre o device token
```swift
UNUserNotificationCenter.current().requestAuthorization(options: [.alert, .badge, .sound]) { granted, _ in
  if granted { DispatchQueue.main.async { UIApplication.shared.registerForRemoteNotifications() } }
}
```

### Configure Android for Push Notifications
- Firebase Messaging (google-services.json)
- Service para onNewToken/onMessageReceived

### Recapping Push Notifications
- Rails guarda tokens
- iOS (APNs) e Android (FCM) recebem e mostram notificações
- Use dados customizados (ex: url) para navegar ao abrir a notificação
