---
title: "Calling RESTful APIs from Flutter using BLoC"
description: "A tutorial for calling RESTful APIs from Flutter"
pubDate: 2021-05-21
tags: ["api", "bloc", "dart", "flutter", "restful"]
---

I’m currently experimenting with [BLoC](https://bloclibrary.dev/#/) on Flutter – a pattern (and library) for managing state. I was trying to find an example for connecting to a REST API and found [an excellent tutorial by Kustiawanto Halim on just that](https://medium.com/flutter-community/flutter-essential-what-you-need-to-know-567ad25dcd8f#5228). Although I’m a huge fan of rapidly advancing technology, unfortunately, it has meant the tutorial is a little out of date. For example, Dart now has its own implementation of the `required` annotation and has made significant changes for null safety. Since the best way to ensure you know something is to teach it, here’s an updated version. I’ve missed out on testing and code coverage for the moment. The completed code can be found at [https://github.com/rob-lowcock/random_quote](https://github.com/rob-lowcock/random_quote)

## Tutorial

### Dependencies

The first thing to do is create our project. Navigate to your chosen folder in your favourite terminal and run the following:

```bash
flutter create random_quote
cd random_quote
```

Now open up `pubspec.yaml` and replace your dependencies block with the following:

```yaml
dependencies:
  flutter:
    sdk: flutter
  flutter_bloc: ^7.0.0
  http: ^0.13.0
  equatable: ^2.0.0
```

A quick explanation of the dependencies we’re adding here:

- **flutter:** Should be pretty self-explanatory!
- **flutter_bloc:**The library we’ll be using for state management.
- **http:**A library for making HTTP connections and requests.
- **equatable:** A useful little library that allows us to compare two different objects, and saves us some boilerplate code. [More info here](https://pub.dev/packages/equatable).

### The API

Like the original tutorial, we’ll be using the Quote Garden API. However, we’ll be using the up-to-date version 3, which has a slightly more nested structure. Specifically, we’ll be hitting https://quote-garden.herokuapp.com/api/v3/quotes/random which gives us this structure:

```json
{
  "statusCode": 200,
  "message": "Random quotes",
  "pagination": {
    "currentPage": 1,
    "nextPage": null,
    "totalPages": 1
  },
  "totalQuotes": 1,
  "data": [
    {
      "_id": "5eb17aaeb69dc744b4e726d0",
      "quoteText": "The principle of all successful effort is to try to do not what is absolutely the best, but what is easily within our power, and suited for our temperament and condition.",
      "quoteAuthor": "John Ruskin",
      "quoteGenre": "best",
      "__v": 0
    }
  ]
}
```

### Data Model

Let’s create the model for a quote. Create a new file at lib/models/quote.dart. This will specify the fields we’re using for a quote, together with some code for how to return that data. Let’s do it in 2 stages. First, create the fields:

```dart
import 'package:equatable/equatable.dart';

class Quote extends Equatable {
  final id;
  final String quoteText;
  final String quoteAuthor;
  const Quote({this.id, required this.quoteText, required this.quoteAuthor});

  @override
  List<Object> get props => [id, quoteText, quoteAuthor];
}
```

To run through what we’re doing here: we’re specifying the id, quoteText, and quoteAuthor as part of our model. Note that in our constructor this.quoteText and this.quoteAuthor are both specified as required (hooray for null safety!).

Next we’re specifying an override for the props. This is to allow equatable to do its thing and allow us to compare two different quotes.

Now let’s expand it a bit further so we can convert a quote from JSON:

```dart
import 'package:equatable/equatable.dart';

class Quote extends Equatable {
  final id;
  final String quoteText;
  final String quoteAuthor;
  const Quote({this.id, required this.quoteText, required this.quoteAuthor});

  @override
  List<Object> get props => [id, quoteText, quoteAuthor];

  static Quote fromJson(dynamic json) {
    return Quote(
      id: json['data'][0]['_id'],
      quoteText: json['data'][0]['quoteText'],
      quoteAuthor: json['data'][0]['quoteAuthor'],
    );
  }

  @override
  String toString() => 'Quote { id: $id }';
}
```

We’ve added two methods here: fromJson() which returns a quote object from a block of JSON, and toString() which allows us to return a basic string from the object. If you’ve followed the original tutorial, you’ll note a change here in what we’re returning from the JSON, as there’s slightly more nesting going on.

Finally, let’s export the model in a barrel. Create a new file lib/models/models.dart, which contains the following:

```dart
export './quote.dart';
```

This lets us access the model more easily. Even though we only have one other file in the folder, it is a good habit to get into (as noted in the original tutorial).

### Data Provider

Now we’ll create the QuoteApiClient, which retrieves deals with the HTTP connection. Create a new file at lib/repositories/quote_api_client.dart. Here’s our starting point:

```dart
class QuoteApiClient {
  final _baseUrl = 'https://quote-garden.herokuapp.com/api/v3';
  final http.Client httpClient;
  QuoteApiClient({
    required this.httpClient,
  });
}
```

A couple of things to note here: we’ve updated the _baseUrl to the new version 3 of the API, and we’re taking advantage of Dart’s new required keyword. We don’t need to do any assertions either, as that will be taken care of for us.

Now let’s add in the fetchQuote method, which will handle hitting the API and retrieving the response. We’ll also add in the import statements.

```dart
import 'dart:convert';

import 'package:http/http.dart' as http;

import 'package:random_quote/models/models.dart';

class QuoteApiClient {
  final _baseUrl = 'https://quote-garden.herokuapp.com/api/v3';
  final http.Client httpClient;
  QuoteApiClient({
    required this.httpClient,
  });

  Future<Quote> fetchQuote() async {
    final url = '$_baseUrl/quotes/random';
    final response = await this.httpClient.get(Uri.parse(url));

    if (response.statusCode != 200) {
      throw new Exception('error getting quote');
    }

    final json = jsonDecode(response.body);
    return Quote.fromJson(json);
  }
}
```

Let’s run through this bit-by-bit. fetchQuote() is an async function, as we’re waiting for a response from the API. This lets us use the await keyword when running this.httpClient.get(). We have to use Uri.parse() to convert the string URL into something the get() method can use.

We check that we’ve got a 200 (status OK) response, then finally decode the response body into JSON and call our fromJson() method.

### Data Repository

Let’s create another file in the repositories folder – this time lib/repositories/quote_repository.dart. The point of this file is to create a layer that wraps around our API calls. This is slightly artificial in this example, as it only calls the quote_api_client.dart methods, but in real-world usage it allows you to separate calling the API and handling the data you get from it. For example, you might have several different API calls.

```dart
import 'package:random_quote/repositories/quote_api_client.dart';

import 'dart:async';

import 'package:random_quote/models/models.dart';

class QuoteRepository {
  final QuoteApiClient quoteApiClient;

  QuoteRepository({required this.quoteApiClient});

  Future<Quote> fetchQuote() async {
    return await quoteApiClient.fetchQuote();
  }
}
```

Finally, let’s create our barrel export again (partly because it’s good practice, and partly because I really like that “barrel export” is a technical term here and not just talking about beer). Note that we’re not exporting quote_api_client.dart, because that should only be used by repositories.

```dart
export './quote_repository.dart';
```

### Business Logic and BLoC

Finally onto the BLoC! QuoteBloc receives QuoteEvents and converts them to QuoteStates. It depends on QuoteRepository so that it can retrieve the quote.

#### QuoteEvent

The quote event is the first thing to create, and also the simplest. Let’s create lib/bloc/quote_event.dart. There’s only one event – fetching the quote.

```dart
import 'package:equatable/equatable.dart';

abstract class QuoteEvent extends Equatable {
  const QuoteEvent();
}

class FetchQuote extends QuoteEvent {
  const FetchQuote();

  @override
  List<Object> get props => [];
}
```

This file looks a little odd at the moment, in that it doesn’t call anything we’ve already written and doesn’t reference the triggering event. Don’t worry too much – we’ll add the event when we first fetch a quote.

#### QuoteState

There are 4 possible states we’ll track inside the QuoteState:

1. QuoteEmpty – we haven’t yet got a quote
2. QuoteLoading – we’re getting a quote, but it hasn’t arrived yet
3. QuoteLoaded – we have our quote
4. QuoteError – something went horribly wrong when we tried to get our quote

The only one of these we’ll do much with is QuoteLoaded, as we need the actual quote to supply it to the view. Each possible state is defined as a class, extending an abstract class of QuoteState. All of this lives in lib/bloc/quote_state.dart

```dart
import 'package:equatable/equatable.dart';

import 'package:random_quote/models/models.dart';

abstract class QuoteState extends Equatable {
  const QuoteState();

  @override
  List<Object> get props => [];
}

class QuoteEmpty extends QuoteState {}

class QuoteLoading extends QuoteState {}

class QuoteLoaded extends QuoteState {
  final Quote quote;

  const QuoteLoaded({required this.quote});

  @override
  List<Object> get props => [quote];
}

class QuoteError extends QuoteState {}
```

#### Quote Bloc

Now we get to the main point of BLoC – converting our event to a state. Let’s take a look at the code of lib/bloc/quote_bloc.dart first, then go through it.

```dart
import 'package:bloc/bloc.dart';
import 'package:random_quote/bloc/quote_event.dart';
import 'package:random_quote/bloc/quote_state.dart';

import 'package:random_quote/repositories/repository.dart';
import 'package:random_quote/models/models.dart';
import 'package:random_quote/bloc/bloc.dart';

class QuoteBloc extends Bloc<QuoteEvent, QuoteState> {
  final QuoteRepository repository;

  QuoteBloc({required this.repository}) : super(QuoteEmpty());

  @override
  Stream<QuoteState> mapEventToState(QuoteEvent event) async* {
    if (event is FetchQuote) {
      yield QuoteLoading();
      try {
        final Quote quote = await repository.fetchQuote();
        yield QuoteLoaded(quote: quote);
      } catch (_) {
        yield QuoteError();
      }
    }
  }
}
```

There’s only one major change from the original tutorial here – [initialState is no longer overridden](https://github.com/felangel/bloc/issues/1304). Instead, we specify the initial state in the constructor with the call to super(QuoteEmpty()).

Then, we override mapEventToState(), which is where our logic comes in. We ordinarily yield QuoteLoading(), and then wait for our quote to load from the repository before yielding QuoteLoaded() (along with our actual quote). If anything goes wrong, then we yield QuoteError().

Note that the QuoteBloc constructor requires the repository. This is where we inject the repository so we can call fetchQuote().

#### Barrel Export

Once again, we barrel export the relevant files as lib/bloc/bloc.dart

```dart
export './quote_event.dart';
export './quote_state.dart';
export './quote_bloc.dart';
```

### Bringing everything together in main.dart

Now we can bring everything together. You can ditch the contents of main.dart and replace it with this:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:http/http.dart' as http;
import 'package:bloc/bloc.dart';

import 'package:random_quote/bloc/bloc.dart';
import 'package:random_quote/repositories/quote_api_client.dart';
import 'package:random_quote/repositories/repository.dart';
import 'package:random_quote/views/home_page.dart';

class SimpleBlocObserver extends BlocObserver {
  @override
  void onTransition(Bloc bloc, Transition transition) {
    super.onTransition(bloc, transition);
    print(transition);
  }
}

void main() {
  Bloc.observer = SimpleBlocObserver();

  final QuoteRepository repository = QuoteRepository(
    quoteApiClient: QuoteApiClient(
      httpClient: http.Client(),
    ),
  );

  runApp(App(
    repository: repository,
  ));
}
```

Our SimpleBlocObserver just prints whenever there’s a transition between states and gives us the details. (As an aside: this was previously SimpleBlocDelegate, but BlocDelegate is now deprecated in favour of observers). We then add our observer in main(), along with setting up our repository (and injecting the QuoteApiClient and http.Client).

The final thing to do in main.dart is to create the app itself. The final file looks like this:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:http/http.dart' as http;
import 'package:bloc/bloc.dart';

import 'package:random_quote/bloc/bloc.dart';
import 'package:random_quote/repositories/quote_api_client.dart';
import 'package:random_quote/repositories/repository.dart';
import 'package:random_quote/views/home_page.dart';

class SimpleBlocObserver extends BlocObserver {
  @override
  void onTransition(Bloc bloc, Transition transition) {
    super.onTransition(bloc, transition);
    print(transition);
  }
}

void main() {
  Bloc.observer = SimpleBlocObserver();

  final QuoteRepository repository = QuoteRepository(
    quoteApiClient: QuoteApiClient(
      httpClient: http.Client(),
    ),
  );

  runApp(App(
    repository: repository,
  ));
}

class App extends StatelessWidget {
  final QuoteRepository repository;

  App({Key? key, required this.repository}) : super(key: key);

  // This widget is the root of your application.
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Quote App',
      home: Scaffold(
        appBar: AppBar(title: Text('Quote')),
        body: BlocProvider(
          create: (context) => QuoteBloc(repository: repository),
          child: HomePage(),
        ),
      ),
    );
  }
}
```

BlocProvider is a useful widget that allows us to inject a BLoC (in our case QuoteBloc) into a child widget. If we wanted to pass multiple BLoCs we could use MultiBlocProvider.

There will be 2 errors if you’ve got linting enabled: package:random_quote/views/home_page.dart won’t be found, and so the HomePage() function won’t exist. We’ll sort these out next.

### Creating the View

We’ll be returning a BlocBuilder, which in turn holds a builder function that returns a different widget depending on the state (e.g. a progress indicator if we’re still loading). All this happens in lib/views/home_page.dart

```dart
import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';

import 'package:random_quote/bloc/bloc.dart';

class HomePage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return BlocBuilder<QuoteBloc, QuoteState>(
      builder: (context, state) {
        if (state is QuoteEmpty) {
          BlocProvider.of<QuoteBloc>(context).add(FetchQuote());
        }
        if (state is QuoteError) {
          return Center(
            child: Text('failed to fetch quote'),
          );
        }
        if (state is QuoteLoaded) {
          return ListTile(
            leading: Text(
              '${state.quote.id}',
              style: TextStyle(fontSize: 10.0),
            ),
            title: Text(state.quote.quoteText),
            isThreeLine: true,
            subtitle: Text(state.quote.quoteAuthor),
            dense: true,
          );
        }
        return Center(
          child: CircularProgressIndicator(),
        );
      },
    );
  }
}
```

### Running the app

Finally we can run the app and, fingers crossed, everything should work.

```bash
flutter run
```

### Next steps

The next steps in the original tutorial involve testing the code, but the next thing I want to work out is how to refresh the quote by pressing a button.
