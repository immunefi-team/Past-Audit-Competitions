# #47741 \[W\&A-Insight] Missing JWT\_SECRET in Env Allows Token Forgery via Empty Secret

**Submitted on Jun 19th 2025 at 13:03:19 UTC by @adhd for** [**IOP | Zano Trade**](https://immunefi.com/audit-competition/iop-zano-trade)

* **Report ID:** #47741
* **Report Type:** Websites & Apps
* **Report severity:** Insight
* **Target:** <https://github.com/PRavaga/zano-p2p/blob/master/api/controllers/auth.controller.ts>
* **Impacts:**

## Description

## Brief/Intro

The jwt.sign and jwt.verify uses `process.env.JWT_SECRET || ""` in the signing and verifying process in this if by any reason if JWT\_SECRET is missing in the env of the project the code will automatically use the empty string as the secret which is a bad practice

## Vulnerability Details

If JWT\_SECRET doesn't exists in the .env then the sign and verify will use empty string causing the token forgery

## Impact Details

Any user can create an valid signature if the JWT\_SECRET is not presnet in the .env and call the verification wrt to the user

## Proof of Concept

## Proof of Concept

```
if (success) {
				const token = jwt.sign(
					{ ...userData },
					process.env.JWT_SECRET || '',
					neverExpires ? undefined : { expiresIn: '24h' },
				);
				res.status(200).send({ success, data: token });
			}
```

```
async verifyToken(req: Request, res: Response, next: NextFunction) {
		try {
			const userData = jwt.verify(req.body.token, process.env.JWT_SECRET || '') as UserData;
			req.body.userData = userData;
			next();
		} catch {
			res.status(401).send({ success: false, data: 'Unauthorized (JWT)' });
		}
	}
```

```
try {
		userData = jwt.verify(data.token, process.env.JWT_SECRET || '') as UserData;
	} catch {
		return next(new Error('Unauthorized'));
	}
```

```
try {
			userData = jwt.verify(data.token, process.env.JWT_SECRET || '') as UserData;
		} catch {
			return next(new Error('Unauthorized'));
		}
```

Here, you can see the use of `||`, which should not be present.
