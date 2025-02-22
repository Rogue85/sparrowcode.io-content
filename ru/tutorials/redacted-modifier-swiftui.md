В iOS 14 и SwiftUI 2 добавили модификатор `.redacted(reason:)`, с помощью которого можно сделать прототип вью:

```swift
VStack {
    Label("Swift Playground", systemImage: "swift")
    Label("Swift Playground", systemImage: "swift")
        .redacted(reason: .placeholder)
}
```

![Прототип вью.](https://cdn.sparrowcode.io/tutorials/redacted-modifier-swiftui/redacted_placeholder.jpg)

Когда пригодится прототип:

1. Показать вью, контент которой будет доступен после загрузки.
2. Показать недоступное или частично доступное содержимое.
3. Использовать вместо `ProgressView()`, о которой я [рассказал в гайде](https://sparrowcode.io/ru/mastering-progressview-swiftui).

Давайте рассмотрим сложный пример:

```swift
struct Device {

    let name: String
    let systemIcon: String
    let description: String
}

extension Device {

    static let airTag: Self =
        .init(
            name: "AirTag",
            systemIcon: "airtag",
            description: "Суперлёгкий способ находить свои вещи. Прикрепите один трекер AirTag к ключам, а другой — к рюкзаку. И теперь их видно на карте в приложении «Локатор»."
        )
}
```

У модели есть название, системная иконка и описание. Я вынес `airTag` в расширение, а сейчас мы создадим отдельную вью:

```swift
struct DeviceView: View {
    let device: Device
    
    var body: some View {
        VStack(spacing: 20) {
            HStack {
                Image(systemName: device.systemIcon)
                    .resizable()
                    .frame(width: 42, height: 42)
                Text(device.name)
                    .font(.title2)
            }
            VStack {
                Text(device.description)
                    .font(.footnote)
                
                Button("Перейти к покупке") {}
                .buttonStyle(.bordered)
                .padding(.vertical)
            }
        }
        .padding(.horizontal)
    }
}
```

Добавляем `DeviceView` в основную вью:

```swift
struct ContentView: View {

    var body: some View {
        DeviceView(device: .airTag)
            .redacted(reason: .placeholder)
    }
}
```

![Результат `DeviceView`.](https://cdn.sparrowcode.io/tutorials/redacted-modifier-swiftui/redacted_deviceview.jpg)

Слева вью без модификатора. Справа — с ним. Давайте для наглядности добавим переключатель:

```swift
struct ContentView: View {

    @State private var toggleRedacted: Bool = false
    
    var body: some View {
        VStack {
            DeviceView(device: .airTag)
                .redacted(reason: toggleRedacted ? .placeholder : [])
            
            Toggle("Toggle redacted", isOn: $toggleRedacted)
                .padding()
        }
    }
}
```

[Переключение между вью с модификатором и без, с помощью переключателя.](https://cdn.sparrowcode.io/tutorials/redacted-modifier-swiftui/redacted_toggle.mov)

## Unredacted

Если вы хотите не скрывать контент, примените модификатор `unredacted()`:

```swift
VStack(spacing: 20) {
    HStack {
        Image(systemName: device.systemIcon)
            .resizable()
            .frame(width: 42, height: 42)
        Text(device.name)
            .font(.title2)
    }
    .unredacted()
    
    VStack {
        Text(device.description)
            .font(.footnote)
            // Какой-то код ниже
```

![Отображение с `Unredacted`.](https://cdn.sparrowcode.io/tutorials/redacted-modifier-swiftui/redacted_unredacted.jpg)

В примере иконка и название девайса не скрыты.

## Кликабельность

Кнопка остаётся кликабельной и работает даже после того, как применили модификатор:

```swift
VStack {
    Text(device.description)
        .font(.footnote)
    
    Button("Перейти к покупке") {
        print("Кнопка кликабельна!")
    }
    .buttonStyle(.bordered)
    .padding(.vertical)
}
```

[Работа кнопки после применения модификатора.](https://cdn.sparrowcode.io/tutorials/redacted-modifier-swiftui/redacted_available_button.mov)

Поведением кнопки управляйте вручную, ниже покажу, как это сделать.

## Причины редактирования

Apple спроектировала структуру [RedactionReasons](https://developer.apple.com/documentation/swiftui/redactionreasons), которая отвечает за **причину** редактирования, применяемую ко вью.
Есть варианты `privacy` и `placeholder`. Первый отвечает за данные, которые скрыты как приватная информация, а placeholder отвечает за обобщённый прототип.

Как можно реализовать кастомную причину:

```swift
extension RedactionReasons {

	static let name = RedactionReasons(rawValue: 1 << 20)
	static let description = RedactionReasons(rawValue: 2 << 20)
}
```

Реализуем с помощью протокола `OptionSet`.

## Environment

У окружения есть проперти `\.redactionReasons` — текущая причина редактирования, применяемая к иерархии вью. Изменим `DevicesView` с помощью `unredacted(when:)`:

```swift
struct DeviceView: View {

    let device: Device
    @Environment(\.redactionReasons) var reasons 
    
    var body: some View {
        VStack(spacing: 20) {
            HStack {
                Image(systemName: device.systemIcon)
                    .resizable()
                    .frame(width: 42, height: 42)
                Text(device.name)
                    .unredacted(when: !reasons.contains(.name))
                    .font(.title2)
            }
            
            VStack {
                Text(device.description)
                    .unredacted(when: !reasons.contains(.description))
                    .font(.footnote)
                
                Button("Перейти к покупке") {
                    print("Кнопка не кликабельна!")
                }
                .disabled(!reasons.isEmpty)
                .buttonStyle(.bordered)
                .padding(.vertical)
            }
        }
        .padding(.horizontal)
    }
}
```

Я добавил кастомный метод `unredacted(when:)` для демонстрации свойства `reasons`:

```swift
extension View {
    @ViewBuilder
    func unredacted(when condition: Bool) -> some View {
        switch condition {
        case true: unredacted()
        case false: redacted(reason: .placeholder)
        }
    }
}
```

Если переключить, кнопка станет некликабельной.

![Кастомный `unredacted`.](https://cdn.sparrowcode.io/tutorials/redacted-modifier-swiftui/redacted_custom_unredacted.jpg)

## Собственный API

Начнём с реализации своих причин:

```swift
enum Reasons {

    case blurred
    case standart
    case sensitiveData
}
```

Реализуем вью-модификаторы, подходящие под причины выше:

```swift
struct Blurred: ViewModifier {

    func body(content: Content) -> some View {
        content
            .padding()
            .blur(radius: 4)
            .background(.thinMaterial, in: Capsule())
    }
}

struct Standart: ViewModifier {

    func body(content: Content) -> some View {
        content
            .padding()
    }
}

struct SensitiveData: ViewModifier {

    func body(content: Content) -> some View {
        VStack {
            Text("Are you over 18 years old?")
                .bold()
            
            content
                .padding()
                .frame(width: 160, height: 160)
                .overlay(.black, in: RoundedRectangle(cornerRadius: 20))
        }
    }
}
```

Чтобы увидеть результат из модификаторов выше в live preview, нужен код:

```swift
struct Blurred_Previews: PreviewProvider {

    static var previews: some View {
        Text("Hello, world!")
            .modifier(Blurred())
    }
}
```

![Отображение с `Blurred`-модификатором.](https://cdn.sparrowcode.io/tutorials/redacted-modifier-swiftui/redacted_blurred_previews.jpg)

Я взял `Blurred`-модификатор. Перейдём к следующему модификатору вью `RedactableModifier`:

```swift
struct RedactableModifier: ViewModifier {

    let reason: Reasons?
    
    init(with reason: Reasons) { self.reason = reason }
    
    @ViewBuilder
    func body(content: Content) -> some View {
        switch reason {
        case .blurred: content.modifier(Blurred())
        case .standart: content.modifier(Standart())
        case .sensitiveData: content.modifier(SensitiveData())
        case nil: content
        }
    }
}
```

У структуры есть `reason`-свойство, которое принимает опциональное перечисление `Reasons`.

Последний шаг — реализовать метод к протоколу `View`:

```swift
extension View {

    func redacted(with reason: Reasons?) -> some View {
        modifier(RedactableModifier(with: reason ?? .standart))
    }
}
```

Я не стал делать отдельную вью, в которой буду вызывать модификаторы. Вместо этого поместил всё в live preview:

```swift
struct RedactableModifier_Previews: PreviewProvider {

    static var previews: some View {
        VStack(spacing: 30) {
            Text("Usual content")
                .redacted(with: nil)
            Text("How are good your eyes?")
                .redacted(with: .blurred)
            Text("Sensitive data")
                .redacted(with: .sensitiveData)
        }
    }
}
```

Результат:

![Отображение после применения `RedactableModifier`.](https://cdn.sparrowcode.io/tutorials/redacted-modifier-swiftui/redacted_redactable_modifier.jpg)
