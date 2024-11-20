# Propriedade Distinct

A propriedade `distinct` é um novo recurso no ASP v2 que oferece controle sobre quando seu Atom dispara reatividade. Ao criar um Atom, você pode especificar se deseja que ele dispare atualizações mesmo quando receber o mesmo valor de estado.

## Entendendo o Distinct

Por padrão, os Atoms usam comparação distinct (similar ao `==`) para determinar se o estado mudou. Isso significa que se você definir o mesmo valor que o Atom atualmente possui, ele não disparará reatividade. No entanto, podem existir casos em que você deseja forçar atualizações mesmo quando o valor permanece o mesmo.

## Uso

Você pode configurar a propriedade `distinct` ao criar um Atom:

```dart
// Comportamento padrão - não dispara se for o mesmo valor
final counterState = atom(0);

// Vai disparar mesmo com o mesmo valor
final counterState = atom(
  0,
  distinct: false,
);
```

## Exemplos

### Comportamento Padrão (distinct: true)

```dart
final messageState = atom('Olá');

// Cria um efeito para monitorar mudanças
atomEffect(
  (get) => get(messageState),
  (state) => print('Mensagem alterada para: $state'),
);

// Não vai disparar o efeito (mesmo valor)
messageState.setValue('Olá');

// Vai disparar o efeito (valor diferente)
messageState.setValue('Olá Mundo');
```

### Com distinct: false

```dart
final timestampState = atom(
  DateTime.now(),
  distinct: false,
);

// Cria um efeito para monitorar mudanças
atomEffect(
  (get) => get(timestampState),
  (state) => print('Timestamp atualizado: $state'),
);

// Vai disparar o efeito mesmo com o mesmo valor
timestampState.setValue(timestampState.state);
```

## Exemplo Prático com Tipagem

### Dashboard com Dados de Vendas (distinct: true)

```dart
// Definindo um tipo personalizado para os dados
class SalesData {
  final double revenue;
  final int orders;
  final DateTime lastUpdate;
  final List<String> recentTransactions;

  SalesData({
    required this.revenue,
    required this.orders,
    required this.lastUpdate,
    required this.recentTransactions,
  });

  @override
  bool operator ==(Object other) {
    if (identical(this, other)) return true;
    return other is SalesData &&
        other.revenue == revenue &&
        other.orders == orders &&
        other.lastUpdate.isAtSameMomentAs(lastUpdate) &&
        listEquals(other.recentTransactions, recentTransactions);
  }

  @override
  int get hashCode => Object.hash(
        revenue,
        orders,
        lastUpdate,
        Object.hashAll(recentTransactions),
      );

  SalesData copyWith({
    double? revenue,
    int? orders,
    DateTime? lastUpdate,
    List<String>? recentTransactions,
  }) {
    return SalesData(
      revenue: revenue ?? this.revenue,
      orders: orders ?? this.orders,
      lastUpdate: lastUpdate ?? this.lastUpdate,
      recentTransactions: recentTransactions ?? List.from(this.recentTransactions),
    );
  }
}

// Criando um Atom tipado para os dados de vendas
final salesDataState = atom<SalesData>(
  SalesData(
    revenue: 0.0,
    orders: 0,
    lastUpdate: DateTime.now(),
    recentTransactions: [],
  ),
  // Usando distinct: true (padrão) pois implementamos corretamente a comparação
);

// Exemplo de uso em um widget com hooks
class SalesDashboard extends StatelessWidget with HookMixin {
  @override
  Widget build(BuildContext context) {
    // Usando useAtomState ao invés de context.watch
    final salesData = useAtomState(salesDataState);

    // Exemplo de uso do useAtomEffect para logging
    useAtomEffect(
      (get) => get(salesDataState),
      effect: (SalesData data) {
        print('Nova venda registrada! Total: R\$ ${data.revenue}');
      },
    );

    return Column(
      children: [
        Text('Receita: R\$ ${salesData.revenue.toStringAsFixed(2)}'),
        Text('Pedidos: ${salesData.orders}'),
        Text('Última Atualização: ${salesData.lastUpdate}'),
        Expanded(
          child: ListView.builder(
            itemCount: salesData.recentTransactions.length,
            itemBuilder: (context, index) {
              return ListTile(
                title: Text(salesData.recentTransactions[index]),
              );
            },
          ),
        ),
        ElevatedButton(
          onPressed: () {
            final newTransaction = 'Venda #${salesData.orders + 1}';
            
            // Atualizando os dados criando um novo objeto
            salesDataState.setValue(
              salesData.copyWith(
                revenue: salesData.revenue + 100.0,
                orders: salesData.orders + 1,
                lastUpdate: DateTime.now(),
                recentTransactions: [
                  newTransaction,
                  ...salesData.recentTransactions.take(9),
                ],
              ),
            );
          },
          child: Text('Registrar Nova Venda'),
        ),
      ],
    );
  }
}
```

Neste exemplo com `distinct: true`:

1. Implementamos uma comparação de igualdade robusta que:
   - Compara corretamente timestamps usando `isAtSameMomentAs`
   - Compara listas usando `listEquals`
   - Implementa `hashCode` de forma consistente

2. Adicionamos um método `copyWith` para facilitar atualizações imutáveis

3. Incluímos uma lista de transações recentes para demonstrar:
   - Manipulação correta de coleções
   - Atualizações imutáveis de estado
   - Comparação profunda de objetos

4. O Atom só dispara atualizações quando:
   - O valor da receita muda
   - O número de pedidos muda
   - O timestamp é diferente
   - A lista de transações é modificada

Este exemplo demonstra como trabalhar corretamente com objetos complexos mantendo o `distinct: true`, o que é mais eficiente que desabilitar a comparação.

## Casos de Uso

A opção `distinct: false` é particularmente útil em cenários como:

1. Atualizações de timestamp onde você quer forçar uma atualização
2. Gatilhos de atualização onde o valor pode ser o mesmo
3. Listas ou objetos que podem ter mudanças internas não detectadas pelo `==`
4. Cenários de atualização forçada em componentes de UI

## Boas Práticas

- Mantenha `distinct: true` (padrão) a menos que você tenha uma necessidade específica de disparar atualizações com o mesmo valor
- Use `distinct: false` com moderação, pois pode levar a atualizações desnecessárias
- Considere implementar comparações de igualdade personalizadas para objetos complexos em vez de desabilitar o distinct

## Observação

Lembre-se que desabilitar a comparação distinct (`distinct: false`) pode impactar o desempenho se usado extensivamente, pois disparará reatividade em cada atualização de estado, independentemente do valor. Use esse recurso de forma consciente com base no seu caso de uso específico. 