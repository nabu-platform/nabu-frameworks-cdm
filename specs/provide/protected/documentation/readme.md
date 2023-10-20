In the beginning the idea was that the provider simply returns the object how it sees it.
This means it was _not_ passed in in the input.

However, in some cases a provider does not have an explicit opinion about a field but it is hard to express that.
If a provider NEVER has an opinion about the field, it should be evident from the subtype it is using, so it should never be able to unset something.
But sometimes it has sporadic interest in a field, based on whatever internal logic it uses.

If you leave it null, you may want to explicitly unset it or actually leave whatever value there is. Making a hard decision either way excludes the other option.

In the end, the decision was made that we send in the data we already have currently. The provider can then choose to enrich specifically how it sees fit.