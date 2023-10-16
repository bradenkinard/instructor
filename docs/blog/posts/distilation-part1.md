---
draft: False  
date: 2023-10-15
tags:
  - python
  - distillation
  - function calling
  - finetuning
  - experimental
---

# Experimental: Finetuning with `Instructions` from `Instructor`

The core philosophy with the `instructor` library is to make language models backwards compatible with existing code. By adding Pydantic in the mix we're able to easily work with LLMs without much worry.

However, building efficient, reliable function is a key skill in software development. But why stop there? What if your functions could automatically become smarter and more efficient without any hand-holding? That's exactly what you gain by investing a few minutes into this read. Here, we delve into some new features `instructor`. 

!!! note "Experimental"
    This is an experimental feature. It's not yet ready for production use. This post is meant to give you a sneak peek into what's coming next, and get your feedback on what you'd like to see.

By the end of this article, you'll understand how to easily integrate the end to end finetuning of small functions `instructor` library with your Python functions to improve them without breaking existing code.

## Why You Should Care

Traditionally, implementing a complex prompt chain involved linking multiple chains together. Each llm call might need [data validation](https://jxnl.github.io/instructor/reask_validation/), external validations, follow up prompts and more. This can be a tedious process, especially if you're working with a large number of functions. Instead we might want to finetune a model that can handle the entire chain end to

### Anatomy of a Complex Function

To paint a clearer picture, let's consider a function that takes a video transcript and churns out an email. This function may include the following steps:

1. Summarize the video transcript.
2. Fact-check the summary.
3. Create a sequence of increasingly dense email drafts.
4. Select the final draft.

Here's how the function could look in code:

```python
def complex_chain(video_transcript: str) -> Email:
    """
    Generate a follow-up email based on a video transcript
    """
    summary = extract_summary(video_transcript)
    summary = check_for_hallucinations(video_transcript, summary)
    emails: List[Email] = create_chain_of_density(summary)
    return emails[-1]
```

Traditional approaches would require you to manually save logs, extract the logs into a training set, fine-tune the model, and then replace the function with the model. But with `instructor`, a single decorator does the trick.

```python
from instructor import Instructions

instructions = Instructions(name="sales_follow_up")

@instructions.distil
def complex_chain(video_transcript: str) -> Email:
    summary = extract_summary(video_transcript)
    summary = check_for_hallucinations(video_transcript, summary)
    emails: List[Email] = create_chain_of_density(summary)
    return emails[-1]
```

This allows you to replace the function with a fine-tuned model without altering the response type or breaking existing code.

## I trained the model. Now what?

Once a model is trained, you might imagine you want to delete the code body and replace it with a call to the model. However since we already decorate the function with `@instructions.distil`, we can simply call the function as usual. Here, `@distil` will automatically detect the model and use it instead of the function body.

```python
from instructor import Instructions

instructions = Instructions(name="sales_follow_up")

@instructions.distil(model='gpt-3.5-turbo:finetuned')
def complex_chain(video_transcript: str) -> Email:
    summary = extract_summary(video_transcript)
    summary = check_for_hallucinations(video_transcript, summary)
    emails: List[Email] = create_chain_of_density(summary)
    return emails[-1]
```

Its a bit advanced but notice that `@distil` can detect the model and call openai rather than calling the base function:

```python
def distil(model):
    def decorator(fn):
        @wraps(fn)
        def wrapper(*args, **kwargs):
            if model:
                return openai.ChatCompletion.create(
                    model=model,
                    messages=[...],
                    response_model=fn.__annotations__["return"],
                )
            else:
                return fn(*args, **kwargs)
        return wrapper
    return decorator
```

You can imagine in the future we can swap baesd on a feature flag or some other logic like a percentage of users. We can even run both the model and the function body in parallel and compare the results, as a way to validate the model.

## A Simpler Example: Three-Digit Multiplication

Even for trivial functions, defining data transformations and gathering the data can still be tedious. Here's how `instructor` automates this.

```python
import logging
import random
from pydantic import BaseModel
from instructor import Instructions

logging.basicConfig(level=logging.INFO)

instructions = Instructions(
    name="three_digit_multiply",
    finetune_format="messages",
    log_handlers=[logging.FileHandler("math_finetunes.jsonl")]
)

class Multiply(BaseModel):
    a: int
    b: int
    result: int

@instructions.distil
def fn(a: int, b: int) -> Multiply:
    resp = a * b
    return Multiply(a=a, b=b, result=resp)

for _ in range(10):
    a = random.randint(100, 999)
    b = random.randint(100, 999)
    print(fn(a, b))
```

Once your function is defined, you can run it and it will automatically log the output to the file specified in the `log_handlers` argument.

## Logging output

```python
{
    "messages": [
        {"role": "system", "content": 'Predict the results of this function: ...'},
        {"role": "user", "content": 'Return fn(133, b=539)'},
        {"role": "assistant", 
            "function_call": 
                {
                    "name": "Multiply", 
                    "arguments": '{"a":133,"b":539,"result":89509}'
            }
        }
    ],
    "functions": [
        {"name": "Multiply", "description": "Correctly extracted `Multiply`..."}
    ]
}
```

Now this file is ready to be used for finetuning. You can use the `instructor` CLI to finetune the model. Check out the [finetune docs](https://jxnl.github.io/instructor/cli/finetune/) for more information.

## Next step

The `instructor` library offers an effortless way to make your llm functions smarter and more efficient. The best part? It ensures backward compatibility, so you can implement these improvements without breaking your existing codebase. 

Now if you're thinking wow, I'd love a backend service to do this for continously, you're in luck! Check out the survey at [useinstructor.com](https://useinstructor.com) and let us know who you are.