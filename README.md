# Scenetest

_Evaluate your product, not your tests. Write friendly little Scenes for your Actors. A Javascript Testing framework inspired by Playwright's `page.evaluate`_ and React server actions.

---

## Concept

Scenetest began from picking at the idea that end-to-end testing kind of seems to be confusing two very different and important things:

1. Start to Finish (StF): Test a set of user interactions from beginning to end so we can codify key user journeys and flows, edge cases and strange user inputs, and make sure they still work as code changes.
2. Client to Server (CtS): Compare data across different contexts to ensure that what you find in your database matches what shows up on screen, as well as in your browser cache, localStorage, or any other context.

Often times, when you ask someone to define End-to-End testing they will say "testing a workflow from start to finish," and then if you pause and give them time to speak, they will end up adding in something about "comparing the database to the browser". But conceptually these two are _very different_, so we won't be using the term E2E here, we have invented our own little terms called Client to Server and Start to Finish, which we will abbreviate here as CtS and StF.

## Starting Point

We use Playwright in our work because it allows us to test Client-to-Server and Start-to-Finish. We can test what's on the page with `page.locator`, drive interactions forward with `page.click('#my-button')`-style interactions, and then run queries in our superuser context to ensure the database matches with our expectations and assertions. This is really nice!

But having these assertions declared side by side with browser orchestration is confusing, and in today's increasingly local-first meta, you often need to examine not only text on the screen of your simulated users but also the contents of the browser cache or localStorage. So it's not enough to just check that the database has a record and the screen has our updated username on it, we want to look into our local pgLite or localDb or our tanstack/db collections and check that the data is there. What now?

In Playwright you would use `page.evaluate(fn)`, written in your test file and called from node-js, which sends a function to the browser to run in the global context and exfiltrate some data back out to the test server for comparison. Sometimes this is nice and easy, but if the data you need is not available in the global context... this can get messy fast. Here's an example from my language-learning app where users have flash card decks for different languages, held in statically defined `Collections` from Tanstack DB:

```javascript
import { test, expect } from 'playwright'
import { getDeck } from '~test/db-helpers'

const TEST_USER_ID = '1'
const TEST_LANG = 'spa'

test.describe.serial('Create a deck and modify it a few different ways', () => {
	test('1. useNewDeckMutation: create new deck', async ({ page }) => {
		// .. orchestration code to log in & navigate to the add-a-deck page
		// .. orchestration code to create a new deck

		// test that the deck was created in the database
		const deckInDatabase = await getDeck(TEST_USER_ID, TEST_LANG)
		expect(deckInDatabase).toBeTruthy()
		expect(deckInDatabase?.lang).toBe(TEST_LANG)
		expect(deckInDatabase?.uid).toBe(TEST_USER_UID)

		// test that the deck was inserted into local collection
		const deckInCollection = await page.evaluate((lang) => {
			const collection = window.__decksCollection
			return collection?.get(lang)
		}, TEST_LANG)
		expect(deckInCollection).toBeTruthy()
		expect(deckInCollection?.lang).toBe(TEST_LANG)
		expect(deckInCollection?.uid).toBe(TEST_USER_UID)

		// test that the two are the similar enough
		expect(deckInDatabase?.deck_id).toBe(deckInCollection?.deck_id)
		expect(deckInDatabase?.updated_at).toBe(deckInCollection?.updated_at)
	})
	// .. additional tests to perform other actions and check them
})
```

This works, and it gets us to the goal -- multi-context assertions that compare data from Client to Server. But notice the ugly `window.__decksCollection` -- where does this even come from? The answer, dear reader, is that I had to go into my application code and attach the `decksCollection` to the `window` so I can access it from these `page.evaluate` functions. That is... a problem.

And if I wanted to return the data that the actual app is using, say from some line in my component like `const allMyDecks = useDecks()`, I absolutely cannot do it. The closest I can get is I can wire up a bunch of tooling to run this `useDecks()` hook in my evaluation function, but it will not be the same exact result as the one used in the component because I have no way of triggering the evaluation to run using the exact value that I'm expecting in my component code at the exact moment it expects it use it -- which is a _very_ common problem that our tests should be able to help us with.

And, having looked into these solutions, I can only say the level of boilerplate and tooling you have to do, and the amount of code you have to reproduce _inside your tests_ is so much that it feels like you are really just testing your test code only, not your app.

While searching for alternatives that would allow us to get around these limitations, we of course checked out Vitest's In-Source testing, but found that in-source is not the same as in-app and definitely not the same as _in-component_ or _in-scope_. Let's look at their example:

```javascript
// the implementation
export function add(...args: number[]) {
  return args.reduce((a, b) => a + b, 0)
}

// in-source test suites
if (import.meta.vitest) {
  const { it, expect } = import.meta.vitest
  it('add', () => {
    expect(add()).toBe(0)
    expect(add(1)).toBe(1)
    expect(add(1, 2, 3)).toBe(6)
  })
}
```
So here you can see we are able to colocate our expectations with our code, and run them in the same _closure_, but they don't run in the application context.

First off, we should say that Vitest is for more granular unit-testing and this is not a knock against them at all. This in-source testing gets us one step closer to some better ergonomics for testing your logic, but imagine an approach to declaring tests that looked more like this:

```javascript

export function add(...args: number[]) {
  // in-scope test
  hypotheticalTest(() => {
    for(num in args) {
      if (typeof num !== 'number')
        raiseFailure('Oops! Your typescript checks have not prevented you from'
        + ' passing a non-number to the `add` function which only wants numbers')
    }
  })

  // application code
  return args.reduce((a, b) => a + b, 0)
}

```

Here we're testing something very different. This is not real code or an end-to-end test, or even a client-to-server test, but it's starting to scratch at the surface of what changes when you run your test inside the application itself. Now we're testing the mental model of the engineers who maintain the product.

Consider a different question: why do I even need end-to-end tests that are able to compare the screen with the server? Aside from Playwright, most popular test frameworks do not prioritize this multi-context, client-to-server comparison at all. And of course we want to know that the updated username has been persisted to the database, but if that's the only reason, shouldn't we be able to have our simulated user actor simply refresh the page and expect the new username to still be there? Why do we want to check the `updated_at` field at all after making a change, if the user never sees that on the screen?

I think we care about these things because what we're validating with client-to-server tests is not the _user journey_ but the _developers' mental model_ for how the system functions. We are validating implementation details, side effects and triggers and **validating that the system works the way we think it works**.

This is very different from the start-to-finish approach, which cares primarily that the _user_'s experience of the product matches what we set out to build (and hasn't broken with our recent refactor).

## Core Considerations For E2E (StF + CtS)

1. Spec files should be nice and clean -- separating expectation logic from browser orchestration so that our spec files can focus only on the latter: orchestrating user interactions with the product. this allows us to write our scenes from start to finish without having full knowledge of the implentation works.
2. Multi-context evaluations allow us to define expectations about both client and server, comparing data from not only the DOM and the database, but also from component props, stores or collections, a query client or cache.
3. Test the product, not the tests. Evaluations and assertions should compare data gathered from/called by the same functions and in the same contexts as the code they're evaluating. Otherwise, all that duplicated logic in your spec files is not only adding work, but also detracting from the relevance of your test suite.
4. Evaluations live in application code and run passively whenever you're in test mode, whether it's your test suite's user actor or a human being clicking through.

## Show Me Some Code!

Fine! Here.

### In-Scope Test Assertions

```javascript
import { useProfile } from '~lib/hooks'
import { pass } from 'scenetest'

export function ProfileForm() {
  const { data: profile } = useProfile()
  pass(
    'Profile should be available (or null) from the first tick (no "pending" state)',
    profile !== undefined
	)

  if (!profile) return <CreateProfileForm />
  return <EditProfileForm initialData={profile} />
}

```

Please notice: **This test evaluates our mental model for how the code works.** It mimics how you might write a comment in the code, except we've formalised it as a test that will run every time any part of your orchestration suite or your natural clicking-around hits this component. This leads us to our first two major benefits:

<div style="background: rgba(50, 50, 180, 0.3); padding-left: 2rem; padding: 0.5rem 2rem; border-left: 4px solid rgba(50, 50, 180, 0.9); border-radius: 6px;">

**1. We re-use testing logic by running it inside our components and hooks**

You can write 15 different test scenes and whenever they hit this code, they'll run through the same assertions and report results to the test server.
</div>

<div style="background: rgba(50, 50, 180, 0.3); padding-left: 2rem; padding: 0.5rem 2rem; border-left: 4px solid rgba(50, 50, 180, 0.9); border-radius: 6px;">

**2. Our evaluation uses component state (from useProfile) at the  exact same tick in the React component lifecycle that we expect it to be there**

We don't have to do anything to "simulate" the useProfile hook in our testing scripts/specs/scenes; we just use the data we're testing.
</div>

Because the assertions are decoupled from the orchestration scripts, they'll run (and report results) regardless of whether the page is being used by the test-runner UserAgent, or if it's just you, our beloved developer (and target audience), trying to fix this form and visiting `localhost:5173/profile` to see that it looks right.

The `pass` function is stripped out by our Vite plugin so it never runs in production, but when you run in dev or test mode, it evaluates every time this component renders and reports results to the test suite. Having decoupled _Assertion logic_ from _Scene flow_ leads us to the next two really interesting benefits of this approach:

<div style="background: rgba(50, 50, 180, 0.3); padding-left: 2rem; padding: 0.5rem 2rem; border-left: 4px solid rgba(50, 50, 180, 0.9); border-radius: 6px;">

**3. You can improv your own test scenes without ever writing a "spec", just _use the product_**

**4. The person who writes the test scene doesn't have to know how the app works under the hood**

Your PM or a QA manager can freestyle clicking through the app, inputting whatever garbage edge-case user inputs they want, and all your assertions will run in the background and report to the test suite. Writing test scenes doesn't require understanding the intermediate steps, side effects and implementation details of how the app functions under the hood.
</div>

This speaks to one of our biggest concerns with modern test tooling: Test your product, not your tests. Too often tests pass or fail because of implementation details _of the test_. In many cases, confidence in the test suite degrades over time because you have passing tests but a broken product, or failing tests but a working product, and minimising this specific pitfall is one of the primary goals of Scenetest.

### Writing Scenes

So far we've discussed the in-scope, in-app, in-component assertions, but not the scene orchestration that is essential for doing Start-to-Finish tests. Imagine you have written your assertions in the component already, and now it's time to write a scene where a fake user logs in and attempts to edit their username:

```javascript
// ~tests/scenes/edit-profile.spec.ts
import { scene, pass, warn } from 'scenetest'
import { actor1 } from '~tests/actors'
import { setupProfile, getProfile, teardownProfile } from '~tests/utils'
import { loginUser, navigateToOwnProfile, successToast } from '~tests/navigations'

scene.play({
	title: 'The user updates their username and finds it has updated on the page',
	prepareScene: () => setupProfile(actor1.profile),
	cleanupScene: () => teardownProfile(actor1.uid),
	isPrepared: async () => !!(await getProfile(actor1.uid)),
	isCleanedup: async () => !(await getProfile(actor1.uid)),
	sceneFn: async (page) => {
		await page.goto('/login')

		// the dev wrote some re-useable functions for common navigation
		await loginUser(actor1)
		await navigateToOwnProfile()

		const newUsername = 'Scene Da'
		form = page.$('main form')

		form.$('input[name=username]').fill(newUsername)
		await form.submit()

		const newNameInDb = await getProfile(actor1.uid)
		const newNameOnScreen = form.$(`h3#username`).text()

		// testing only outcomes; no side effects or implementation details
		pass(`Username updated successfully in database`, newUsername === newNameInDb)
		pass(`Username updated successfully on screen`, newUsername === newNameOnScreen)
		if (newUsername !== newNameOnScreen) {
			if (newUsername !== newNameInDb) fail('Username failed to update')
			else fail('Username updated in DB but not on screen')
		}
	},
})
```

This code should look strikingly similar to a Playwright spec. And it would work just fine as one. It will fail if you're unable to navigate or log in, if after logging in you're not able to see the username input, if the database entry doesn't match the new username value, or if the screen doesn't update to show the new username.

But imagine a situation where the profile is being stored in a collection, cache or store that doesn't update properly, and the username's `h3` element is written as `<h3>{mutation.data?.username ?? profile.username }</h3>`. In this scenario, the test would pass because the mutation data contains the new username, but if the profile page unmounts and remounts, the old value will still show until we refresh the page, refetch the queries, and rebuild the local cache. This is exactly, _exactly_ the kind of thing we need tests for, and exactly the kind of thing that is so difficult to do with mainstream test tooling.

### Client-to-Server & In-Scope Tests

Now that we've seen how the scene flow works, here's our profile form, with assertions that will test our expectations about how the form works, with assertions that compare data from client and server:

```javascript
// ~components/profile-form.ts
import { assertion, pass, fail } from 'scenetest'
import { useForm } from '@your-fave/form'
import { useStore } from 'zustand'
import { toast } from 'your-fave-toast'
import { updateProfile } from '~lib/my-api'
import { store } from '~lib/my-store'

export function UpdateProfileForm() {
	const profile = useStore(store, data => data.profile)
	fail('ProfileForm rendered before profile is available', !profile)

	const myForm = useForm({
		onSubmit: updateProfile,
		onSuccess: (data, inputs) => {
			const newProfile = ProfileSchema.parse(data)
			store.updateProfile(newProfile)
			toast.show('Updated successfully!')
		},
		onError: () => toast.error('Update failed :('),
		onSettled: {
			assertion({
				title: 'Updating profile'
				// the expectFn provides a "server" object that you configure (see below)
				assertFn: async (server, fromApp) => {
					// wait for the toast before checking everything else
					await pass('shows success toast', /* ... */)

					const dbProfile = await server.getProfile(fromApp.userId)
					pass('has DB updated_at value in the last 10 seconds', /* ... */)
					pass.happy('DB value updated less than 1 second ago', /* ... */)
					pass('has DB values same as form inputs', /* ... */)

					// not all values will match the DB exactly, but these should
					const fieldsToCompare = [
						'username', 'uid', 'language', 'avatar_url', 'bio', 'updated_at',
					]
					pass('has same values in DB & local collection', () => {
						for(key in fieldsToCompare) {
							if (dbProfile[key] !== fromApp.localDbData[key])
								// failure doesn't end the test; it just records a failure
								fail(`Values for "${key}" do not match`)
						}
					})
				},
				appData: () => ({
					userId,
					formSuccessData: newProfile,
					localDbData: store.getProfile(),
					formInputs: inputs,
				}),
			})
		}
	})
	return <form onSubmut={myForm.handleSubmit}>{/* ... */}</form>
}
```

You can think of this `assertFn` as analogous to a _Server Action_ in React parlance (but with no returned data); we're defining it in our component, with access to the react-world's data objects and types via the `appData`, but the assertion function itself will be stripped by the bundler and shipped to run on our test server with elevated permissions for the database and other privileged resources, and will report its results only to the test suite. (You might think of it as a "Server Effect" but that's another conversation!)

This is where we are getting back to the initial concept of `page.evaluate` from Playwright land, except instead of having both the server logic and the browser logic written in the _test_, cloistered away from all the code they're evaluating (and probably treated like a haunted attic by most of your dev team), they live in the component. They produce the `appData` at exactly the spot in the React lifecycle where the form's `onSettled` function is called -- exactly when our mental model of the code tells us it should be ready.

A couple of key things to note here:

1. This assertion will also run when a tester is improvising their own test scenes or clicking randomly throughout the app trying to break things. Updated the profile? The test running will evaluate the appData fn, ship it to the test server, and run the assertFn to make sure everything went as planned.
2. Now that we are fully separating Scene Flow from Assertion Logic, anything that doesn't break the flow of the script doesn't have to end the test. If the flow succeeds but the assertions fail, this tells us "Your product is probably working but your mental model is off". I think this is where people will really love the developer-experience benefits of behind Scenetest; it should actually make your job easier.
3. As promised: our Client-to-Server testing doesn't have to wire up special infrastructure to access props or state of the React component, and we can access the form's inputs directly, rather than hard-coding them into our spec file as `PROFILE_1_UPDATE_PROFILE_OLD_USERNAME` and `PROFILE_1_UPDATE_PROFILE_NEW_USERNAME`.

### Friendly Little Scripts

Now that we've seen the full store from Client to Server and Start to Finish, we should go back and look at the example scene given above. This is the same code, but focusing only on the `sceneFn`:

```javascript
// ~tests/scenes/edit-profile.spec.ts
// .. ignoring imports fttb

scene({
	// .. ignoring title, prepare, cleanup
	sceneFn: async (page) => {
		// .. ignoring login / navigate
		const newUsername = 'Scene Da'
		form = page.$('main form')

		form.$('input[name=username]').fill(newUsername)
		await form.submit()

		const newNameInDb = await getProfile(actor1.uid)
		const newNameOnScreen = form.$(`h3#username`).text()

		// testing only outcomes; no side effects or implementation details
		pass(`Username updated successfully in database`, newUsername === newNameInDb)
		pass(`Username updated successfully on screen`, newUsername === newNameOnScreen)
		if (newUsername !== newNameOnScreen) {
			if (newUsername !== newNameInDb) fail('Username failed to update')
			else fail('Username updated in DB but not on screen')
		}
	},
})
```

When you first saw this, you might have thought it looks really simple because it's an example. That's partly true, but the main reason it looks simple is because it actually is simple. Using another framework, just navigating your app with a test actor can be a technical nightmare, requiring intimate knowledge of your code base and what it all means w/r/t concepts like race conditions, suspense boundaries, network latency, and the React lifecycle. This is nuts, actually!

And further, your orchestration code is going to be intermingled with a thousand different decisions you have to make about whether you should include assertions for this side effect or that one -- "did we already test this side effect in a different spec, or is this different enough that we should test it again?" It's dealer's choice with every one of these tiny decisions along the way, leading to a kind of core conceptual inconsistency that keeps you constantly consulting your senior or delaying PRs while discussions about test coverage run their course.

Our example above works without these awaits and expect-to-be-visibles, and without skimping on any assertion logic, because the vast majority of the awaiting and expecting has already been handled in the in-scope tests. Now, when our user actor in our scene does `await form.submit()`, we know this triggers network requests, promises, and assertions which also trigger round-trips to the test server and back again, and Scenetest knows to await all of that activity before resolving the promise from `form.submit()` and attempting to proceed. In our case, the `assertion` triggers when the form is settled, after the database and the local collection have been updated. And, crucially, the person in charge of the form component was the one who wrote that in that way; the person who wrote the scene didn't have to know any of it.

## Conclusion

### What is Scenetest

It's just a concept doc right now (the one you're reading).

### What are Current Steps

Flesh out the concept, talk to friends.

## Missing From This Doc

1. Actors -- we want to have a formal concept of Actors that make it possible to write one scene where two actors interact with the app. This should allow easier testing of: chat interfaces, friend requests, live editing experiences, and more.
2. Technical Architecture -- documenting the specific architecture of how everything will work. These are mostly solved problems so I haven't bothered yet. Playwright and Tanstack Start exist, so everything in this doc should be doable with existing / proven approaches.
3. Possible ease of testing with browser plugin -- plenty of other systems have browser-plugin approaches where you orchestrate user interactions/scripts by clicking things; this approach could be quite powerful here because if the devs write assertions based on the mental model of the feature they're writing, then anyone on the team can make new scripts by recording point-and-click, giving a ton of extra rigor to the ease-of-use that this "record your session" approach provides.
4. Impact on AI-assisted programming -- it seems quite plausible that this approach will make it easier to use natural language prompts to generate both Scene specs and in-scope Assertions.
	* Scene specs will be easier to write for all the reasons described in this document: they will be far simpler and more readable.
	* Assertions will be easier to write because they will be colocated with the feature code, and the LLM can write them in response to the same prompts. For example: a developer might write a prompt that says, "a user needs to [this thing]. build me a form that shows [this data] and lets me do [this action] to create a new [this item]. after submitting the form the user should see [this message], the database should have [this record] and the app's local storage should have [this item], and the hook that provides state to the component should now return [this state]". In this way, a good prompt that explains to the LLM what features to write is already giving all of the information it would need to write in-scope tests at the same time.