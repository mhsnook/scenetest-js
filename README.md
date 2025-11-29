# Scenetest

_Evaluate your product, not your tests. Write lovely little Scenes for your Actors. A Javascript Testing framework inspired by Playwright's `page.evaluate`._

---

Scenetest began from the idea that end-to-end testing kind of seemds to be describing two things:

1. Start to finish: Test a set of user interactions from start to finish (we call these _Scenes_)
2. Front to back: Make assertions from client to DB and back again; compare what you find across
  different contexts (on the screen, in the database, in localStorage)

We use Playwright in our work because it allows us to do both of these things. It treats your spec files like writing a scene in a play, using `page.click('#my-button')` style interactions to drive the script forward. This is really nice! And it allows you to use the nodejs / superuser context to check information in the database directly, while you check what you see on the screen to make sure your expectations match, which gives us testing from start to finish and front to back.

But having these evaluations live side by side with browser orchestration is confusing, and in today's increasingly local-first development practices, you increasingly need to examine not only text on the screen but also the contents of the browser cache or localStorage. In Playwright you
would use `const someClientData = page.evaluate((data) => clientDataFn, dataFromServer)` to get data out of the browser and into the test context where you can run your evaluation. But if you want your evaluation to use data accessible from inside a react context or a dynamically created store or queryClient, you have to jump through a lot of hoops to make that work.

While searching for alternatives that would allow us to get around this last limitation, we of course checked out Vitest's In-Source testing, but found that in-source is not the same as in-scope. You may colocate your evaluations with your code, but it doesn't run in the application context.

So these are the two core concerns that Scenetest addresses:

1. Keep your spec files nice and clean: just write lovely little scripts for your automated actors to simulate user interactions.
2. Define your expectations in the same code you're evaluating (where they are much easier to read, and can double as documentation of your code)
3. Call your expectations in the same Javascript scope
Colocate your expectations with the code they're evaluating, and call them from in the same scope: Instead of the test context calling `page.evaluate((data) => fn, data)`, the application context defines expectations which call `scenetest((server, data) => fn, data)` when they need to access the data as a superuser or service role.

So your Scene orchestrates the actions, and your in-app expectations run as a consequence of this; they're not defined in the spec, they just run when the app runs in test mode. One benefit of this is you can fire up the app in test mode and use it yourself! So all the assertions are still running while you improv your own scene. Here's an example component using in-app expectations; no spec files, completely un-concerned with which tests are using it:

```javascript
// ~components/profile-form.ts
import { scenetest, expect, failure } from 'scenetest'
import { useStore } from 'zustand'
import { useForm } from '@your-fave/form'
import { useStore } from 'zustand'
import { updateProfile } from '~lib/my-api'
import { store } from '~lib/my-store'

export function ProfileForm() {
	const profile = useStore(store, data => data.profile)
	expect('ProfileForm only renders when profile available first-tick', !!profile)

	const form = useForm({
		onSubmit: updateProfile,
		onSuccess: (data, inputs) => {
			const newProfile = ProfileSchema.parse(data)
			store.updateProfile(newProfile)
			toast.show('Updated successfully!')
		},
		onError: toast.error('Update failed :(')
		onSettled: {
			scenetest({
				title: 'Updating profile'
				// the expectFn provides a "server" object that you configure (see below)
				expectFn: async (server, fromApp) => {
					// wait for the toast before checking everything else
					await expect('shows success toast', /* ... */)

					const dbProfile = await server.getProfile(fromApp.userId)
					expect('has DB updated_at value in the last 10 seconds', /* ... */)
					expect('has DB values same as form inputs', /* ... */)

					// some values may not match the DB exactly, b/c serialisation etc.
					// but the developer felt like these fields definitely should match
					const fieldsToCompare = [
						'username', 'uid', 'language', 'avatar_url', 'bio', 'updated_at',
					]
					expect('has same values in DB & local collection', () => {
						for(key in fieldsToCompare) {
							if (dbProfile[key] !== fromApp.localDbData[key])
								// failure doesn't end the test; it just records a failure
								failure(`Values for "${key}" do not match`)
						}
					})
				},
				appDataFn: () => ({
					userId,
					formSuccessData: newProfile,
					localDbData: store.getProfile(),
					formInputs: inputs,
					// profile,
				}),
				// to test reactivity for a `profile = useProfile()` hook, we might add a tiny delay
				// delay: 50,
			})
		}
	})
	// if (!profile) return <>Loading...</>
	return <UpdateProfileForm form={form} initialData={profile} />
}
```

There's a lot to notice about this example, so let's unpack it a bit.

First, please notice the logical flow of this process:

1. We decide that at a certain part in the application code, we want to test some things that just happened -- we just concluded a mutation, this is maybe the best example of when you want your mutation to run front-to-back, app-to-server and back again. So we declare the `scenetest()` to run on the `onSettled` callback for our form.
2. the `scenetest` config object accepts an `expectFn` which runs in nodejs and an `appDataFn` which runs on the client, evaluates the function, and passes the data to the server for evaluation. This is like Playwright's `page.evaluate` but in reverse.
3. Because of this, we're able to execute `appDataFn` not only colocated with the code its testing, but in the same javascript scope, able to evaluate the store's `getProfile` in the very same tick in the react/javascript lifecycle.
4. We're also able to access the form inputs and `profile` variable directly -- not just the ones we hardcoded into the spec file as `PROFILE_1_UPDATE_PROFILE_OLD_USERNAME` and `PROFILE_1_UPDATE_PROFILE_NEW_USERNAME`, but the profile returned from the hook and the inputs passed to the form.

We're moving a lot of the testing logic into the component, and that's on purpose. And because all the necessary expectations and optional delay and retries would be handled in this code, our orchestration code, our spec files where we define our Scene is going to be much simpler.

```javascript
// ~tests/scenes/profile.spec.ts
import { test, goal, warn } from 'scenetest'
import { actor1 } from '~tests/actors'
import { setupProfile, getProfile, teardownProfile } from '~tests/utils'
import { loginUser, navigateToOwnProfile, successToast } from '~tests/navigations'

test.scene({
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
		const newNameOnScreen = form.$(`input[name=username]`).text()

		// we are only testing final outcomes
		goal(`Username updated successfully in database`, newUsername === newNameInDb)
		goal(`Username updated successfully on screen`, newUsername === newNameOnScreen)
		if (newUsername !== newNameOnScreen) {
			if (newUsername !== newNameInDb) fail('Username failed to update')
			else fail('Username updated in DB but not on screen')
		}
	},
})
```

Here we are, just writing a lovely little script, what the user does and what the user gets.

You might notice there's no asserting happening until the very end. That's because we've separated
the front-to-back evaluations into the in-scope tests above, and here we only need to concern
ourselves with the navigation through the user journey from start to finish. In fact, even the
goal and fail at the end feels out of place.

What if we changed it a little to look like this...

```javascript
// ~tests/scenes/profile.spec.ts
// .. same imports

test.scene({
	// ..
	sceneFn: async (page) => {
		// .. same login + navigate

		const newUsername = 'Scene Da'
		form = page.$('main form')

		form.$('input[name=username]').fill(newUsername)
		await form.submit()

		const newNameOnScreen = form.$(`input[name=username]`).text()

		// we are not testing anything!
		return {
			input: newUsername,
			onscreen: newNameOnScreen,
		}
	},
	reviewFn: ({ input, onscreen }) => {
		const newNameInDb = await getProfile(actor1.uid)

		goal(`Username updated successfully in database`, input === database)
		goal(`Username updated successfully on screen`, input === onscreen)
		if (input !== onscreen) {
			if (input !== database) fail('Username failed to update')
			else fail('Username updated in DB but not on screen')
		}
	},
})
```

Now that feels nice. The `sceneFn` is only orchestrating user interactions, it's not running
any checks at all. It's returning some data that gets passed to the `reviewFn`.

We've also moved the database query from the scene function to the review function, because
we're following the mental model of giving a "test script" to a human person who just does
what's in the script, and we watch and evaluate, and this back-end property of the database
is not visible to actor1 who we have hired (or simulated) to perform our scene.

You may also notice that, now the `sceneFn` doesn't make any attempt to access the database,
and the `reviewFn` doesn't make any attempt to access the `page`. This is an area for further
exploration. So far we've been working with the idea of orchestration vs. evaluation, but
it begs the question, which one does this line belong with:

```javascript
const newNameOnScreen = form.$(`input[name=username]`).text()
```

Clearly the script orchestration was over once the form was submitted... and if we decided to
pass the `page` context to the reviewFn we could always "evaluate" it there... why include it
in the `sceneFn` at all? Food for further thought.

Another one for further thought is -- why include the `reviewFn` at all? If the point of the
scene is to orchestrate user interactions start to finish, and the point of the in-scope tests
is to test the back-to-front data matching, then perhaps the better answer is to leave out the
DB check entirely and just let the scene function return a boolean:

```javascript
// ~tests/scenes/profile.spec.ts
// .. same imports, except no getProfile this time

test.scene({
	// ..
	sceneFn: async (page) => {
		// .. same login + navigate

		const newUsername = 'Scene Da'
		form = page.$('main form')

		form.$('input[name=username]').fill(newUsername)
		await form.submit()

		return (newUsername === form.$(`input[name=username]`).text())
	},
	// no reviewFn needed
})
```

Of course, if you wanted to you could still put goals and fails in there to surface additional
information for reporting, (like in the first example), but the exploratory work for now is to
figure out the smoothest ergonomics for keeping these concepts separate enough and close enough.

I think the core idea here is that we don't want to be testing side effects or testing our tests,
and the primary concept we are using for this is to separate the idea of "end-to-end" into two
axes: back-to-front, and start-to-finish. And this way you don't have to feel like your
script-writing (start-to-finish) code is hampered by having to insert assertions all over the
place, because your back-to-front testing is happening every step of the way, without you having
to say a word.

In this way, we are making parts of our testing process re-usable in exactly the way we already
re-use react components, functions, hooks, etc. -- by moving the F2B tests into those very same
functions and having them run whenever the functions run. All of this together should make the
rigorous parts more rigorous, and the flow-y parts more flow-y. Scripting new "runs" through
your app shouldn't feel awful; it shouldn't take forever. You should be able to just turn on
the recorder and click through your app and have it a) record a new start-to-finish scene script,
(without having to know anything at all about what should be asserted along the way), and b) run
the front-to-back tests as you go and stream the results to your testing UI.
